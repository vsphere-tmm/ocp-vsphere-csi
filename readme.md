# Installing CSI on OCP

## Update the VMs to VMHW 15 and include UUID

### Export GOVC vars

```sh
export GOVC_PASSWORD=P@ssw0rd                                                                                              
export GOVC_USERNAME=administrator@vsphere.local
export GOVC_URL=https://vc01.satm.eng.vmware.com/sdk
export GOVC_INSECURE=1
export GOVC_USERNAME=administrator@vsphere.local
```

### Export OCP cluster name

```sh
export OCP_CLUSTER_NAME=demo
```

### Patch VMs

```sh
# Power off VMs
govc find / -type m -runtime.powerState poweredOn -name $OCP_CLUSTER_NAME-'*' | xargs -L 1 govc vm.power -off $1
# Enable Disk UUID
govc find / -type m -runtime.powerState poweredOff -name $OCP_CLUSTER_NAME-'*' | xargs -L 1 govc vm.change -e="disk.enableUUID=1" -vm $1
# Upgrade VMHW to v15
govc find / -type m -runtime.powerState poweredOff -name $OCP_CLUSTER_NAME-'*' | xargs -L 1 govc vm.upgrade -version=15 -vm $1
# Power on VMs
govc find / -type m -runtime.powerState poweredOff -name $OCP_CLUSTER_NAME-'*' | grep -v rhcos | xargs -L 1 govc vm.power -on $1
```

## Taint nodes for CPI install

```sh
kubectl taint nodes --all 'node.cloudprovider.kubernetes.io/uninitialized=true:NoSchedule'
```

## Create secrets and configmaps for CPI and CSI

### Edit the conf files

Open [csi-vsphere.conf](csi-vsphere.conf) and change `cluster-id` so that it is unique _**in your vCenter**_, using the OCP cluster ID, i.e: `demo-qrtnt` would be adequate.

**N.B: It is _extremely_ important that this is unique per K8s cluster, or you will have volume mounting problems. I.E: Each K8s cluster should have a totally unique `cluster-id`.**

Also edit the `vCenter address`, `username`, `password`, `datacenter` to your environment - do the same for [vsphere.conf](vsphere.conf).

### Apply conf files to cluster

```sh
oc create secret generic vsphere-config-secret --from-file=csi-vsphere.conf --namespace=kube-system
oc get secret vsphere-config-secret --namespace=kube-system
oc create configmap cloud-config --from-file=vsphere.conf --namespace=kube-system
```

## Install CPI
Official Documentation - [Install vSphere Cloud Provider Interface](https://docs.vmware.com/en/VMware-vSphere-Container-Storage-Plug-in/2.0/vmware-vsphere-csp-getting-started/GUID-0C202FC5-F973-4D24-B383-DDA27DA49BFA.html)

```sh
oc apply -f https://raw.githubusercontent.com/kubernetes/cloud-provider-vsphere/master/manifests/controller-manager/cloud-controller-manager-roles.yaml
oc apply -f https://raw.githubusercontent.com/kubernetes/cloud-provider-vsphere/master/manifests/controller-manager/cloud-controller-manager-role-bindings.yaml
oc apply -f https://github.com/kubernetes/cloud-provider-vsphere/raw/master/manifests/controller-manager/vsphere-cloud-controller-manager-ds.yaml
```

### Verify CPI

This must print out a `ProviderID` per node, or CSI will not work - if it is not populated, then CPI was probably not initialised correctly (check your taints).

```sh
kubectl describe nodes | grep "ProviderID"
```

## Install CSI
Official Documentation - [Deploy the vSphere Container Storage Plug-in on a Native Kubernetes Cluster](https://docs.vmware.com/en/VMware-vSphere-Container-Storage-Plug-in/2.0/vmware-vsphere-csp-getting-started/GUID-A1982536-F741-4614-A6F2-ADEE21AA4588.html)
### For vSphere 6.7 U3 and later

````sh
# Taint the Control Plane nodes
kubectl taint nodes <k8s-primary-name> node-role.kubernetes.io/master=:NoSchedule

# Create the namespace
oc apply -f https://github.com/kubernetes-sigs/vsphere-csi-driver/blob/release-2.4/manifests/vanilla/namespace.yaml

# Install the CSI
oc apply -f https://raw.githubusercontent.com/kubernetes-sigs/vsphere-csi-driver/v2.4.0/manifests/vanilla/vsphere-csi-driver.yaml
````

### Verify CSI

```sh
kubectl get deployment --namespace=vmware-system-csi
kubectl get daemonsets vsphere-csi-node --namespace=vmware-system-csi
kubectl describe csidrivers
kubectl get CSINode
```

## Create a StorageClass and Deploy a volume

Edit the [sc.yaml](./sc.yaml) and [pvc.yaml](./pvc.yaml) to suit your environment (change the storage policy in the SC, adjust the name if desired).

```sh
kubectl apply -f sc.yaml
kubectl apply -f pvc.yaml
```

### Verify the deployed volume

Output in the `STATUS` column, after 10-15s should be `Bound` for both:

```sh
kubectl get pv,pvc
```
