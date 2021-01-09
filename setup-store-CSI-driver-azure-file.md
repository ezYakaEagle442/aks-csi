See also :

- [Azure Storage accounts](https://docs.microsoft.com/en-us/azure/storage/common/storage-account-overview?toc=%2Fazure%2Fstorage%2Ffiles%2Ftoc.json#types-of-storage-accounts)
- [Azure Storage availability in your region](https://azure.microsoft.com/en-gb/global-infrastructure/services/?regions=france-central,france-south,europe-west,europe-north&products=storage)
- [https://docs.microsoft.com/en-us/azure/storage/files](https://docs.microsoft.com/en-us/azure/storage/files)
- [https://docs.microsoft.com/en-us/azure/aks/azure-files-dynamic-pv](https://docs.microsoft.com/en-us/azure/aks/azure-files-dynamic-pv)
- [https://docs.microsoft.com/en-us/azure/aks/azure-files-volume](https://docs.microsoft.com/en-us/azure/aks/azure-files-volume)
- [https://github.com/container-storage-interface/spec](https://github.com/container-storage-interface/spec)
- [https://github.com/kubernetes-sigs/azurefile-csi-driver](https://github.com/kubernetes-sigs/azurefile-csi-driver)
- [https://kubernetes-csi.github.io/docs/topology.html](https://kubernetes-csi.github.io/dks/topology.html)
- [https://kubernetes-csi.github.io/docs/drivers.html](https://kubernetes-csi.github.io/dks/drivers.html)

# Pre-req

See :
- [install guide](https://github.com/kubernetes-sigs/azurefile-csi-driver/blob/master/docs/install-azurefile-csi-driver.md)
- Available [sku](https://github.com/kubernetes-sigs/azurefile-csi-driver/blob/master/docs/driver-parameters.md) are : Standard_LRS, Standard_ZRS, Standard_GRS, Standard_RAGRS, Premium_LRS
- [Pre-req](https://github.com/kubernetes-sigs/azurefile-csi-driver#prerequisite) : The driver initialization depends on a Cloud provider config file.

The driver initialization depends on a Cloud provider config file, usually it's /etc/kubernetes/azure.json on all kubernetes nodes deployed by AKS or aks-engine, here is azure.json example. This driver also supports read cloud config from kuberenetes secret.

## Azure Cloud Provider

The step below is not mandatory as the config should be fetch sucessfully in AKS from /etc/kubernetes/azure.json

```sh

# https://github.com/kubernetes-sigs/azurefile-csi-driver/blob/master/docs/read-from-secret.md
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

[Azure Files](https://docs.microsoft.com/en-us/azure/aks/azure-files-dynamic-pv) supports Premium storage in AKS clusters, minimum premium file share is **100GB**.

```sh
# https://docs.microsoft.com/en-us/azure/role-based-access-control/built-in-roles#managed-identity-operator
# https://docs.microsoft.com/en-us/azure/role-based-access-control/built-in-roles#virtual-machine-contributor
az role assignment create --role "Managed Identity Operator" --assignee $aks_client_id --scope /subscriptions/$subId/resourcegroups/$managed_rg
az role assignment create --role "Virtual Machine Contributor" --assignee $aks_client_id --scope /subscriptions/$subId/resourcegroups/$managed_rg

az role assignment create --role "Contributor" --assignee $aks_client_id --scope /subscriptions/$subId/resourcegroups/$rg_name
az role assignment create --role "Contributor" --assignee $aks_client_id --scope /subscriptions/$subId/resourcegroups/$managed_rg

```

# Install the Azure File CSI Driver

```sh

driver_version=master #vv0.6.0
echo "Driver version " $driver_version
curl -skSL https://raw.githubusercontent.com/kubernetes-sigs/azurefile-csi-driver/$driver_version/deploy/install-driver.sh | bash -s $driver_version --

k get rolebinding -n kube-system | grep -i "azurefile"
k get role -n kube-system | grep -i "azurefile"
k get ClusterRoleBinding | grep -i "azurefile"
k get ClusterRole | grep -i "azurefile"
k get cm -n kube-system  | grep -i "azurefile"
k get sa -n kube-system | grep -i "azurefile"
k get svc -n kube-system
k get psp | grep -i "azurefile"
k get ds -n kube-system | grep -i "azurefile"
k get deploy -n kube-system | grep -i "azurefile"
k get rs -n kube-system | grep -i "azurefile"
k get po -n kube-system | grep -i "azurefile"
k get sc -A

k describe clusterrole csi-azurefile-controller-secret-role
k describe clusterrolebinding csi-azurefile-controller-secret-binding

# k get pod -n kube-system -l app=csi-azurefile-controller -o wide --watch 
# k get pod -n kube-system -l app=csi-azurefile-node -o wide --watch 

k get events -n kube-system | grep -i "Error" 

for pod in $(k get pods -l app=csi-azurefile-controller -n kube-system -o custom-columns=:metadata.name)
do
    k describe pod $pod -n kube-system | grep -i "Error"
    k logs $pod -c csi-provisioner -n kube-system | grep -i "Error"
    k logs $pod -c csi-attacher -n kube-system | grep -i "Error"
    k logs $pod -c csi-snapshotter -n kube-system | grep -i "Error"
    k logs $pod -c csi-resizer -n kube-system | grep -i "Error"
    k logs $pod -c liveness-probe -n kube-system | grep -i "Error"
    k logs $pod -c azurefile -n kube-system | grep -i "Error"
done

for pod in $(k get pods -l app=csi-azurefile-node -n kube-system -o custom-columns=:metadata.name)
do
    k describe pod $pod -n kube-system | grep -i "Error"
    k logs $pod -c liveness-probe -n kube-system #| grep -i "Error"
    k logs $pod -c node-driver-registrar # | grep -i "Error"
    k logs $pod -c azurefile -n kube-system # | grep -i "Error"
done


```
### [Troubleshoot](https://github.com/kubernetes-sigs/azurefile-csi-driver/blob/master/dks/csi-debug.md)

If the logs show failed to get Azure Cloud Provider, ***error: Failed to load config from file: /etc/kubernetes/azure.json***, cloud not get azure cloud provider
it means that you have the cloud provider config file is not correctly set at /etc/kubernetes/azure.json in AKS, or not correctly parametered in the driver yaml file as explained in the [pre-req](#Pre-req)


## Test Azure File CSI Driver
See examples :
- [basic usage](https://github.com/kubernetes-sigs/azurefile-csi-driver/blob/master/deploy/example/e2e_usage.md)

### Option 1: Dynamic Provisioning
```sh

k create -f https://raw.githubusercontent.com/kubernetes-sigs/azurefile-csi-driver/master/deploy/example/storageclass-azurefile-csi.yaml
k create -f https://raw.githubusercontent.com/kubernetes-sigs/azurefile-csi-driver/master/deploy/example/pvc-azurefile-csi.yaml

k get sc -A
k describe sc file.csi.azure.com
```

### Option 2: Static Provisioning (use an existing azure file share)
```sh
# k create -f https://raw.githubusercontent.com/kubernetes-sigs/azurefile-csi-driver/master/deploy/example/storageclass-azurefile-existing-share.yaml

# Create an Azure File
str_name="stfrfile""${appName,,}"

# Premium Files are not available in all Azure Regions,
# See : https://github.com/kubernetes-sigs/azurefile-csi-driver/issues/291
# az storage account create --name $str_name --kind FileStorage --sku Premium_LRS --location $location -g $rg_name 
az storage account create --name $str_name --kind StorageV2 --sku Standard_LRS --location $location -g $rg_name 
az storage account list -g $rg_name

fs_share_name=aksfs
az storage share create --name $fs_share_name --account-name $str_name
az storage share list --account-name $str_name
az storage share show --name $fs_share_name --account-name $str_name

# https://dks.microsoft.com/en-us/azure/storage/files/storage-how-to-use-files-linux
httpEndpoint=$(az storage account show --name $str_name -g $rg_name --query "primaryEndpoints.file" | tr -d '"')
smbPath=$(echo $httpEndpoint | cut -c7-$(expr length $httpEndpoint))$fs_share_name
storageAccountKey=$(az storage account keys list --account-name $str_name -g $rg_name --query "[0].value" | tr -d '"')

echo "httpEndpoint" $httpEndpoint 
echo "smbPath" $smbPath 
echo "storageAccountKey" $storageAccountKey 

export RESOURCE_GROUP=$rg_name
export STORAGE_ACCOUNT_NAME=$str_name
export SHARE_NAME=$fs_share_name

envsubst < ./cnf/storageclass-azurefile-existing-share.yaml > deploy/storageclass-azurefile-existing-share.yaml
cat deploy/storageclass-azurefile-existing-share.yaml

k create -f ./deploy/storageclass-azurefile-existing-share.yaml
k create -f https://raw.githubusercontent.com/kubernetes-sigs/azurefile-csi-driver/master/deploy/example/pvc-azurefile-csi-static.yaml
            
k describe sc file.csi.azure.com

```

### Test Volume Mount
```sh
# validate PVC status and create an nginx pod
k describe pvc pvc-azurefile
k create -f https://raw.githubusercontent.com/kubernetes-sigs/azurefile-csi-driver/master/deploy/example/nginx-pod-azurefile.yaml
k get events -n kube-system | grep -i "Error" 

# enter the pod container to do validation
k describe po nginx-azurefile
k exec -it nginx-azurefile -- bash
# run the command below inside the container
c # /mnt/azurefile directory should mounted as File System
cat  /mnt/azurefile/outfile
```

## Snapshot

- [PV Snapshot](https://kubernetes-csi.github.io/docs/snapshot-restore-feature.html) is supported (beta since k8s v1.17)
- [https://github.com/kubernetes-sigs/azurefile-csi-driver/tree/master/deploy/example/snapshot](https://github.com/kubernetes-sigs/azurefile-csi-driver/tree/master/deploy/example/snapshot)
- [https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates)

To be tested !
```sh

# cat /etc/systemd/system/kubelet.service
# cat /etc/default/kubelet
# cat /var/lib/kubelet/kubeconfig

az snapshot create --name PV1Snapsho --source $pv1 -g $rg_name

```

# Clean-Up
```sh
az storage share delete --name $fs_share_name --account-name $str_name
az storage account delete --name $str_name -g $rg_name -y

k delete pod nginx-azurefile
k delete pvc pvc-azurefile
k delete pv pv-azurefile
k delete sc file.csi.azure.com
curl -skSL https://raw.githubusercontent.com/kubernetes-sigs/azurefile-csi-driver/$driver_version/deploy/uninstall-driver.sh | bash -s --

```
