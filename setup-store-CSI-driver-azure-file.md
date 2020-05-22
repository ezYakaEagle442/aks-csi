See also :

- [https://docs.microsoft.com/en-us/azure/aks/azure-files-dynamic-pv](https://docs.microsoft.com/en-us/azure/aks/azure-files-dynamic-pv)
- [https://docs.microsoft.com/en-us/azure/aks/azure-files-volume](https://docs.microsoft.com/en-us/azure/aks/azure-files-volume)
- [https://dks.openshift.com/aro/4/storage/understanding-persistent-storage.html#types-of-persistent-volumes_understanding-persistent-storage](https://dks.openshift.com/aro/4/storage/understanding-persistent-storage.html#types-of-persistent-volumes_understanding-persistent-storage)
- [https://dks.openshift.com/aro/4/storage/persistent_storage/persistent-storage-azure-file.html](https://dks.openshift.com/aro/4/storage/persistent_storage/persistent-storage-azure-file.htmle)
- [https://github.com/container-storage-interface/spec](https://github.com/container-storage-interface/spec)
- [https://github.com/kubernetes-sigs/azurefile-csi-driver](https://github.com/kubernetes-sigs/azurefile-csi-driver)
- [https://kubernetes-csi.github.io/dks/topology.html](https://kubernetes-csi.github.io/dks/topology.html)
- [https://kubernetes-csi.github.io/dks/drivers.html](https://kubernetes-csi.github.io/dks/drivers.html)

# Pre-req

See :
- [install guide](https://github.com/kubernetes-sigs/azurefile-csi-driver/blob/master/dks/install-azurefile-csi-driver.md)
- Available [sku](https://github.com/kubernetes-sigs/azurefile-csi-driver/blob/master/dks/driver-parameters.md) are : Standard_LRS, Standard_ZRS, Standard_GRS, Standard_RAGRS, Premium_LRS
- [Pre-req](https://github.com/kubernetes-sigs/azurefile-csi-driver#prerequisite) : The driver initialization depends on a Cloud provider config file.

The driver initialization depends on a Cloud provider config file, usually it's /etc/kubernetes/azure.json on all kubernetes nodes deployed by AKS or aks-engine, here is azure.json example. This driver also supports read cloud config from kuberenetes secret.

```sh

# https://github.com/kubernetes-sigs/azurefile-csi-driver/blob/master/dks/read-from-secret.md
mkdir deploy
tenantId=$(az account show --query tenantId -o tsv)

# https://kubernetes.io/dks/concepts/configuration/secret/#decoding-a-secret
k get secrets -n kube-system
k describe secret azure-cloud-provider -n kube-system
azure_cnf_secret=$(k get secret azure-cloud-provider -n kube-system -o jsonpath="{.data.cloud-config}" | base64 --decode)
echo "Azure Cloud Provider config secret " $azure_cnf_secret

azure_cnf_secret_length=$(echo -n $azure_cnf_secret | wc -c)
echo "Azure Cloud Provider config secret length " $azure_cnf_secret_length

aadClientId="${azure_cnf_secret:14:35}"
echo "aadClientId " $aadClientId

aadClientSecret="${azure_cnf_secret:53:$azure_cnf_secret_length}"
echo "aadClientSecret" $aadClientSecret

echo -e "{\n"\
"\""tenantId\"": \""$tenantId\"",\n"\
"\""subscriptionId\"": \""$subId\"",\n"\
"\""resourceGroup\"": \""$rg_name\"",\n"\
"\""useManagedIdentityExtension\"": false,\n"\
"\""aadClientId\"": \""$aadClientId\"",\n"\
"\""aadClientSecret\"": \""$aadClientSecret\""\n"\
"}\n"\
> deployazure.json

cat deploy/azure.json

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
it means that you have the cloud provider config file is not correctly set at /etc/kubernetes/azure.json in AKS, or not correctly paramtered in the driver yaml file as explained in the [pre-req](#Pre-req)


## Test Azure File CSI Driver
See examples :
- [basic usage](https://github.com/kubernetes-sigs/azurefile-csi-driver/blob/master/deploy/example/e2e_usage.md)


```sh
# Option 1: Dynamic Provisioning
k create -f https://raw.githubusercontent.com/kubernetes-sigs/azurefile-csi-driver/master/deploy/example/storageclass-azurefile-csi.yaml
k create -f https://raw.githubusercontent.com/kubernetes-sigs/azurefile-csi-driver/master/deploy/example/pvc-azurefile-csi.yaml

# Option 2: Static Provisioning(use an existing azure file share)
# k create -f https://raw.githubusercontent.com/kubernetes-sigs/azurefile-csi-driver/master/deploy/example/storageclass-azurefile-existing-share.yaml

# Create an Azure File
str_name="stwefile""${appName,,}"
az storage account create --name $str_name --kind FileStorage --sku Premium_ZRS --lkation $lkation -g $rg_name 
az storage account list -g $rg_name

fs_share_name=arofs
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

k create -f ./cnf/storageclass-azurefile-existing-share.yaml
k create -f https://raw.githubusercontent.com/kubernetes-sigs/azurefile-csi-driver/master/deploy/example/pvc-azurefile-csi.yaml

# validate PVC status and create an nginx pod
k describe pvc pvc-azurefile -w
k create -f https://raw.githubusercontent.com/kubernetes-sigs/azurefile-csi-driver/master/deploy/example/nginx-pod-azurefile.yaml

# enter the pod container to do validation
k describe po nginx-azurefile -w
k exec -it nginx-azurefile -- bash
# /mnt/azurefile directory should be mounted as cifs filesystem

```

## Clean-Up
```sh
az storage share delete --name $fs_share_name --account-name $str_name
az storage account delete --name $str_name -g $rg_name -y

k delete sc file.csi.azure.com
k delete pvc pvc-azurefile
k delete pv pv-azurefile
k delete pods xxx

curl -skSL https://raw.githubusercontent.com/kubernetes-sigs/azurefile-csi-driver/$driver_version/deploy/uninstall-driver.sh | bash -s --

```
