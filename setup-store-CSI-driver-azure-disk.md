See also :

- [Azure Storage accounts](https://docs.microsoft.com/en-us/azure/storage/common/storage-account-overview?toc=%2Fazure%2Fstorage%2Ffiles%2Ftoc.json#types-of-storage-accounts)
- [Azure Storage availability in your region](https://azure.microsoft.com/en-gb/global-infrastructure/services/?regions=france-central,france-south,europe-west,europe-north&products=storage)
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


An Azure disk can only be mounted to a single pod at a time. If you need to share a persistent volume across multiple pods, use Azure Files.

## Azure Cloud Provider

The step below is not mandatory as the config should be fetch sucessfully in AKS from /etc/kubernetes/azure.json

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

## Perform role assignments

[When you create an Azure disk for use with AKS](https://docs.microsoft.com/en-us/azure/aks/azure-disk-volume#create-an-azure-disk), you can create the disk resource in the node resource group. This approach allows the AKS cluster to access and manage the disk resource. If you instead create the disk in a separate resource group, you must grant the Azure Kubernetes Service (AKS) service principal for your cluster the Contributor role to the disk's resource group. Alternatively, you can use the system assigned managed identity for permissions instead of the service principal. For more information, see Use managed identities.

```sh
# https://docs.microsoft.com/en-us/azure/role-based-access-control/built-in-roles#managed-identity-operator
# https://docs.microsoft.com/en-us/azure/role-based-access-control/built-in-roles#virtual-machine-contributor
az role assignment create --role "Managed Identity Operator" --assignee $aks_client_id --scope /subscriptions/$subId/resourcegroups/$managed_rg
az role assignment create --role "Virtual Machine Contributor" --assignee $aks_client_id --scope /subscriptions/$subId/resourcegroups/$managed_rg

# does NOT work :  az role assignment create --role "Contributor" --assignee $aks_client_id --scope /subscriptions/$subId/resourcegroups/$rg_name
az role assignment create --role "Contributor" --assignee $aks_client_id --scope /subscriptions/$subId/resourcegroups/$managed_rg

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

## Test Azure Disk CSI Driver

See examples :
- [basic usage](https://github.com/kubernetes-sigs/azuredisk-csi-driver/blob/master/deploy/example/e2e_usage.md)

### Option 1: Azuredisk Dynamic Provisioning
```sh

k create -f https://raw.githubusercontent.com/kubernetes-sigs/azuredisk-csi-driver/master/deploy/example/storageclass-azuredisk-csi.yaml
k create -f https://raw.githubusercontent.com/kubernetes-sigs/azuredisk-csi-driver/master/deploy/example/pvc-azuredisk-csi.yaml

```

### Option 2: Azuredisk Static Provisioning (use an existing azure disk)
```sh
# wget https://raw.githubusercontent.com/kubernetes-sigs/azuredisk-csi-driver/master/deploy/example/pv-azuredisk-csi.yaml > ./cnf/pv-static-azuredisk-csi.yaml

# Create a disk
disk_name="aks-dsk"
az disk create --name $disk_name --location $location -g $managed_rg --sku Premium_LRS --size-gb 5 # --zone 1 
az disk list -g $managed_rg
disk_id=$(az disk show --name $disk_name -g $managed_rg --query id)

export SUBSCRIPTION_ID=$subId
export RESOURCE_GROUP=$managed_rg
export TENANT_ID=$tenantId
export DISK_ID=$disk_name

envsubst < ./cnf/pv-static-azuredisk-csi.yaml > deploy/pv-static-azuredisk-csi.yaml
cat deploy/pv-static-azuredisk-csi.yaml

k create -f deploy/pv-static-azuredisk-csi.yaml
k create -f https://raw.githubusercontent.com/kubernetes-sigs/azuredisk-csi-driver/master/deploy/example/pvc-azuredisk-csi-static.yaml

```

### Test Volume Mount

```sh
# make sure pvc is created and in Bound status finally
k get pvc -o wide
k get pv -o wide

vmss_name=$(az vmss list -g $managed_rg --query [0].name -o tsv)
echo "VMSS name: " $vmss_name

# az vmss show --name $vmss_name -g $managed_rg 
az vmss get-instance-view --name $vmss_name -g $managed_rg

node0_name=$(az vmss list-instances --name $vmss_name -g $managed_rg --query [0].name -o tsv)
echo "Node0 VM name: " $node0_name

# create a pod with azuredisk CSI PVC
k create -f https://raw.githubusercontent.com/kubernetes-sigs/azuredisk-csi-driver/master/deploy/example/nginx-pod-azuredisk.yaml

# enter the pod container to do validation: watch the status of pod until its Status changed from Pending to Running and then enter the pod container
k get po -w
watch k describe po nginx-azuredisk

k exec -it nginx-azuredisk -- bash
# run the command below inside the container
df -k # /mnt/azuredisk directory should mounted as disk filesystem
cat  /mnt/azuredisk/outfile

```

### [Troubleshoot](https://github.com/kubernetes-sigs/azuredisk-csi-driver/blob/master/dks/csi-debug.md)

If the logs show failed to get Azure Cloud Provider, ***error: Failed to load config from file: /etc/kubernetes/azure.json***, cloud not get azure cloud provider
it means that you have the cloud provider config file is not correctly set at /etc/kubernetes/cloud.conf in ARO or /etc/kubernetes/azure.json in AKS, or not correctly paramtered in the driver yaml file as explained in the [pre-req](#Pre-req)


If you received this kind of error, you may not have performed the role assignments: 

***error in vm.get.request: resourceID: /subscriptions/xxxxx/resourceGroups/rg-strcsi-francecentral/providers/Microsoft.Compute/virtualMachines/aks-strcsiaksnp-14664217-vmss000000, error: Retriable: false, RetryAfter: 0s, HTTPStatusCode: 403, RawError: Retriable: false, RetryAfter: 0s, HTTPStatusCode: 403, RawError: {"error":{"code":"AuthorizationFailed","message":"The client 'aaaaaaaaa' with object id 'bbbbbbbbbb' does not have authorization to perform action **'Microsoft.Compute/virtualMachines/read'** over scope '/subscriptions/xxxxx/resourceGroups/rg-strcsi-francecentral/providers/Microsoft.Compute/virtualMachines/aks-strcsiaksnp-14664217-vmss000000' or the scope is invalid. If access was recently granted, please refresh your credentials."}}***

If you received this kind of error, you may not have performed the role assignments: 

 ***HTTPStatusCode: 403, RawError: {"error":{"code":"AuthorizationFailed","message":"The client 'aaaaaaaaa' with object id 'bbbbbbbbbb' does not have authorization to perform action 'Microsoft.Compute/disks/read' over scope '/subscriptions/xxxxxs/resourceGroups/rg-strcsi-francecentral/providers/Microsoft.Compute/disks/aks-dsk' or the scope is invalid. If access was recently granted, please refresh your credentials."}}***

## Snapshot

[PV Snapshot](https://kubernetes-csi.github.io/docs/snapshot-restore-feature.html) is supported (beta since k8s v1.17)


To be tested !
```sh
k get pvc azure-managed-disk
NAME                 STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS      AGE
azure-managed-disk   Bound     pvc-XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXX      5Gi        RWO            managed-premium   3m

$pv1=`az disk list --query '[].id | [?contains(@,`pvc-XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXX`)]' -o tsv`
/subscriptions/<guid>/resourceGroups/MC_MYRESOURCEGROUP_MYAKSCLUSTER_EASTUS/providers/MicrosoftCompute/disks/kubernetes-dynamic-pvc-XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXX

az snapshot create --name PV1Snapsho --source $pv1 -g $rg_name

```

##  Topology (Availability Zone)
[Topology](https://kubernetes-csi.github.io/docs/topology.html) (beta in k8s v1.16, GA in v1.17) is supported : it provides the ability for Kubernetes (users or components) to influence where a volume is provisioned (e.g. provision new volume in either "zone 1" or "zone 2").


```sh
#  : https://github.com/kubernetes-sigs/azuredisk-csi-driver/tree/master/deploy/example/topology
# Check node topology after driver installation
k get no --show-labels | grep topo
```

##  Volume expansion
Volume Expansion / Resizing (beta since k8s v1.16) 
```sh
TODO
```

##  Volume Limits
Restriction on the number of volumes that can be used in a Node (GA in v1.17) : supported
See [https://kubernetes.io/docs/concepts/storage/storage-limits/#kubernetes-default-limits](https://kubernetes.io/docs/concepts/storage/storage-limits/#kubernetes-default-limits)

```sh
TODO
```
##  Shared Disk
Shared Disk (Multi-node ReadWrite): in Alpha, [Premium_LRS only sku, currently only supported in the West Central US region](https://docs.microsoft.com/en-us/azure/virtual-machines/linux/disks-shared#premium-ssds)
```sh
TODO
```

## CSI on Windows support 

(Alpha feature since k8s v1.18)

```sh
TODO
```

# Clean-Up

```sh

k delete pod nginx-azuredisk
k delete pvc pvc-azuredisk
k delete pv pv-azuredisk

az disk delete --name $disk_name -g $rg_name -y


# Uninstall Driver : 
curl -skSL https://raw.githubusercontent.com/kubernetes-sigs/azuredisk-csi-driver/$driver_version/deploy/uninstall-driver.sh | bash -s $driver_version --
k delete StorageClass disk.csi.azure.com
k delete pvc pvc-azuredisk

# Shared disk(Multi-node ReadWrite) , still in Alpha : https://github.com/kubernetes-sigs/azuredisk-csi-driver/tree/master/deploy/example/sharedisk

```
