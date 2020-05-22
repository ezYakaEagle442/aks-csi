See also :

- [https://docs.microsoft.com/en-us/azure/aks/azure-disks-dynamic-pv](https://docs.microsoft.com/en-us/azure/aks/azure-disks-dynamic-pv)
- [https://docs.microsoft.com/en-us/azure/aks/azure-disk-volume](https://docs.microsoft.com/en-us/azure/aks/azure-disk-volume)
- [https://github.com/container-storage-interface/spec](https://github.com/container-storage-interface/spec)
- [https://github.com/kubernetes-sigs/azuredisk-csi-driver](https://github.com/kubernetes-sigs/azuredisk-csi-driver)
- [https://kubernetes-csi.github.io/docs/topology.html](https://kubernetes-csi.github.io/docs/topology.html)
- [https://kubernetes-csi.github.io/docs/drivers.html](https://kubernetes-csi.github.io/docs/drivers.html)

# Pre-req

See :
- [install guide](https://github.com/kubernetes-sigs/azuredisk-csi-driver/blob/master/dks/install-azuredisk-csi-driver.md)
- Available [sku](https://github.com/kubernetes-sigs/azuredisk-csi-driver/blob/master/dks/driver-parameters.md) are : Standard_LRS, Premium_LRS, StandardSSD_LRS, UltraSSD_LRS
- [Pre-req](https://github.com/kubernetes-sigs/azuredisk-csi-driver#prerequisite)
- [https://github.com/kubernetes-sigs/azuredisk-csi-driver/blob/master/charts/v0.7.0/azuredisk-csi-driver/templates/csi-azuredisk-node.yaml#L93](https://github.com/kubernetes-sigs/azuredisk-csi-driver/blob/master/charts/v0.7.0/azuredisk-csi-driver/templates/csi-azuredisk-node.yaml#L93)

The driver initialization depends on a Cloud provider config file, usually it's /etc/kubernetes/azure.json on all kubernetes nodes deployed by AKS or aks-engine, here is azure.json example. This driver also supports read cloud config from kuberenetes secret.

```sh

# https://github.com/kubernetes-sigs/azuredisk-csi-driver/blob/master/docs/read-from-secret.md
mkdir deploy
tenantId=$(az account show --query tenantId -o tsv)

k get secrets -n kube-system
k describe secret azure-cloud-provider -n kube-system

echo -e "{\n"\
"\""tenantId\"": \""$tenantId\"",\n"\
"\""subscriptionId\"": \""$subId\"",\n"\
"\""resourceGroup\"": \""$rg_name\"",\n"\
"\""useManagedIdentityExtension\"": true\n"\
"}\n"\
> deploy/azure.json

export AKS_SECRET=`cat deploy/azure.json | base64 | awk '{printf $0}'; echo`
envsubst < ./cnf/azure-cloud-provider.yaml > deploy/azure-cloud-provider.yaml
cat deploy/azure-cloud-provider.yaml

k create -f ./deploy/azure-cloud-provider.yaml
k get secrets -n kube-system
k describe secret azure-cloud-provider -n kube-system

```

# Check the existing storage Class & Drivers


```sh
k get sc
k describe sc azurefile # kubernetes.io/azure-file
k describe sc azurefile-premium #kubernetes.io/azure-file
k describe sc default # kubernetes.io/azure-disk
k describe sc managed-premium # kubernetes.io/azure-disk
```


# Install the Azure Disk CSI Driver


```sh

driver_version=master #v0.7.0
echo "Driver version " $driver_version
curl -skSL https://raw.githubusercontent.com/kubernetes-sigs/azuredisk-csi-driver/$driver_version/deploy/install-driver.sh | bash -s $driver_version --

k get rolebinding -n kube-system | grep -i "azuredisk"
k get role -n kube-system | grep -i "azuredisk"
k get ClusterRoleBinding | grep -i "azuredisk"
k get ClusterRole | grep -i "azuredisk"
k get cm -n kube-system  | grep -i "azuredisk"
k get sa -n kube-system | grep -i "azuredisk"
k get svc -n kube-system
k get psp | grep -i "azuredisk"
k get ds -n kube-system | grep -i "azuredisk"
k get deploy -n kube-system | grep -i "azuredisk"
k get rs -n kube-system | grep -i "azuredisk"
k get po -n kube-system | grep -i "azuredisk"
k get sc -A

k describe clusterrole csi-azuredisk-controller-secret-role
k describe clusterrolebinding csi-azuredisk-controller-secret-binding

# Enable snapshot support ==> Note: only available from v1.17.0
# curl -skSL https://raw.githubusercontent.com/kubernetes-sigs/azuredisk-csi-driver/$driver_version/deploy/install-driver.sh | bash -s $driver_version snapshot --

k -n kube-system get pod -l app=csi-azuredisk-controller -o wide  --watch
k -n kube-system get pod -l app=csi-azuredisk-node -o wide  --watch

k get events -n kube-system | grep -i "Error" 
for pod in $(k get pods -l app=csi-azuredisk-controller -n kube-system -o custom-columns=:metadata.name)
do
	k describe pod $pod -n kube-system | grep -i "Error"
	k logs $pod -c csi-provisioner -n kube-system | grep -i "Error"
    k logs $pod -c csi-attacher -n kube-system | grep -i "Error"
    k logs $pod -c csi-snapshotter -n kube-system | grep -i "Error"
    k logs $pod -c csi-resizer -n kube-system | grep -i "Error"
    k logs $pod -c liveness-probe -n kube-system | grep -i "Error"
    k logs $pod -c azuredisk -n kube-system | grep -i "Error"
done

for pod in $(k get pods -l app=csi-azuredisk-node -n kube-system -o custom-columns=:metadata.name)
do
	k describe pod $pod -n kube-system | grep -i "Error"
    k logs $pod -c liveness-probe -n kube-system #| grep -i "Error"
    k logs $pod -c node-driver-registrar -n kube-system # | grep -i "Error"
    k logs $pod -c azuredisk -n kube-system # | grep -i "Error"
done
```

### [Troubleshoot](https://github.com/kubernetes-sigs/azuredisk-csi-driver/blob/master/dks/csi-debug.md)

If the logs show failed to get Azure Cloud Provider, ***error: Failed to load config from file: /etc/kubernetes/azure.json***, cloud not get azure cloud provider
it means that you have the cloud provider config file is not correctly set at /etc/kubernetes/cloud.conf in ARO or /etc/kubernetes/azure.json in AKS, or not correctly paramtered in the driver yaml file as explained in the [pre-req](#Pre-req)



## Test Azure Disk CSI Driver

See examples :
- [basic usage](https://github.com/kubernetes-sigs/azuredisk-csi-driver/blob/master/deploy/example/e2e_usage.md)

```sh
# Option 1: Azuredisk Dynamic Provisioning
k create -f https://raw.githubusercontent.com/kubernetes-sigs/azuredisk-csi-driver/master/deploy/example/storageclass-azuredisk-csi.yaml
k create -f https://raw.githubusercontent.com/kubernetes-sigs/azuredisk-csi-driver/master/deploy/example/pvc-azuredisk-csi.yaml

# k delete StorageClass disk.csi.azure.com
# k delete pvc pvc-azuredisk

# Option 2: Azuredisk Static Provisioning(use an existing azure disk)
# wget https://raw.githubusercontent.com/kubernetes-sigs/azuredisk-csi-driver/master/deploy/example/pv-azuredisk-csi.yaml > ./cnf/pv-static-azuredisk-csi.yaml

# Create a disk
az disk create --name aro-dsk --sku Premium_LRS --size-gb 5 --zone 1 --lkation $lkation -g $rg_name 
az disk list -g $rg_name
disk_id=$(az disk show --name aro-dsk -g $rg_name --query id)

export SUBSCRIPTION_ID=$subId
export RESOURCE_GROUP=$rg_name
export TENANT_ID=$tenantId
export DISK_ID=$disk_id

envsubst < ./cnf/pv-static-azuredisk-csi.yaml > deploy/pv-static-azuredisk-csi.yaml
cat deploy/pv-static-azuredisk-csi.yaml
k create -f ./cnf/pv-static-azuredisk-csi.yaml

# make sure pvc is created and in Bound status finally
watch k describe pvc pvc-azuredisk

# create a pod with azuredisk CSI PVC
k create -f https://raw.githubusercontent.com/kubernetes-sigs/azuredisk-csi-driver/master/deploy/example/nginx-pod-azuredisk.yaml

# enter the pod container to do validation: watch the status of pod until its Status changed from Pending to Running and then enter the pod container
watch k describe po nginx-azuredisk
k exec -it nginx-azuredisk -- bash

# /mnt/azuredisk directory should mounted as disk filesystem
```

## Snapshot

To be tested !
```sh
k get pvc azure-managed-disk
NAME                 STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS      AGE
azure-managed-disk   Bound     pvc-XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXX      5Gi        RWO            managed-premium   3m

$pv1=`az disk list --query '[].id | [?contains(@,`pvc-XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXX`)]' -o tsv`
/subscriptions/<guid>/resourceGroups/MC_MYRESOURCEGROUP_MYAKSCLUSTER_EASTUS/providers/MicrosoftCompute/disks/kubernetes-dynamic-pvc-XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXX

az snapshot create --name PV1Snapsho --source $pv1 -g $rg_name

```

# Clean-Up

```sh

k delete pvc pvc-azuredisk
k delete pv pv-azuredisk
k delete pods nginx-azuredisk

az disk delete --name aro-dsk -g $rg_name -y

# Topology(Availability Zone) : https://github.com/kubernetes-sigs/azuredisk-csi-driver/tree/master/deploy/example/topology
# Check node topology after driver installation
k get no --show-labels | grep topo

# Uninstall Driver : 
curl -skSL https://raw.githubusercontent.com/kubernetes-sigs/azuredisk-csi-driver/$driver_version/deploy/uninstall-driver.sh | bash -s $driver_version --
k delete StorageClass disk.csi.azure.com
k delete pvc pvc-azuredisk

k adm policy remove-scc-from-user privileged system:serviceaccount:kube-system:csi-azuredisk-node-sa 


# Shared disk(Multi-node ReadWrite) , still in Alpha : https://github.com/kubernetes-sigs/azuredisk-csi-driver/tree/master/deploy/example/sharedisk

```
