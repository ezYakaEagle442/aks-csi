# Setup Velero

See :
- [https://github.com/vmware-tanzu/velero/blob/master/site/docs/master/supported-providers.md](https://github.com/vmware-tanzu/velero/blob/master/site/docs/master/supported-providers.md)
- [https://github.com/vmware-tanzu/velero-plugin-for-microsoft-azure](https://github.com/vmware-tanzu/velero-plugin-for-microsoft-azure)
- [https://velero.io/docs/v1.3.2/supported-providers/](https://velero.io/docs/v1.3.2/supported-providers/)



<span style="color:red">/!\ IMPORTANT </span> : 

## Pre-requisites

```sh

# Create the storage account.

AZURE_STORAGE_ACCOUNT_ID="velero$(uuidgen | cut -d '-' -f5 | tr '[A-Z]' '[a-z]')"
echo "AZURE_STORAGE_ACCOUNT_ID" $AZURE_STORAGE_ACCOUNT_ID

az storage account create \
    --name $AZURE_STORAGE_ACCOUNT_ID \
    --kind BlobStorage \
    --sku Standard_GRS \
    --encryption-services blob \
    --https-only true \
    --resource-group $rg_name \
    --access-tier Hot

BLOB_CONTAINER=velero
az storage container create -n $BLOB_CONTAINER --public-access off --account-name $AZURE_STORAGE_ACCOUNT_ID

velero_sp_password=$(az ad sp create-for-rbac --name velero-$appName --role Contributor --query password -o tsv)
# velero_sp_password=`az ad sp create-for-rbac --name "velero" --role "Contributor" --query 'password' -o tsv \
#  --scopes /subscriptions/$subId]`

echo "Velero Service Principal PWD" $velero_sp_password


#velero_sp_id=$(az ad sp list --all --query "[?appDisplayName=='velero'].{appId:appId}" --output tsv)
#velero_sp_id=$(az ad sp list --show-mine --query "[?appDisplayName=='velero'].{appId:appId}" --output tsv)
velero_sp_id=$(az ad sp list --display-name velero-$appName --query '[0].appId' -o tsv)
echo "Velero Service Principal ID:" $velero_sp_id 
echo $velero_sp_id > velero_sp_id.txt
# velero_sp_id=`cat velero_sp_id.txt`
az ad sp show --id $velero_sp_id

# az ad sp credential reset -n xxx

# Now you need to create a file that contains all the relevant environment variables. The command looks like the following:

tenantId=$(az account show --query tenantId -o tsv)

aks_client_id=$(az aks show -g $rg_name -n $cluster_name --query identityProfile.kubeletidentity.clientId -o tsv)
echo "AKS Cluster Identity Client ID " $aks_client_id

cat << EOF > ./credentials-velero
AZURE_SUBSCRIPTION_ID=${subId}
AZURE_TENANT_ID=${tenantId}
AZURE_CLIENT_ID=${aks_client_id}
AZURE_CLIENT_SECRET=${velero_sp_password}
AZURE_RESOURCE_GROUP=${rg_name}
AZURE_CLOUD_NAME=AzurePublicCloud
EOF

cat ./credentials-velero

```

## Download Velero

```sh
# https://velero.io/docs/v1.3.2/basic-install/

wget https://github.com/vmware-tanzu/velero/releases/download/v1.3.2/velero-v1.3.2-linux-amd64.tar.gz
tar -zxvf velero-v1.3.2-linux-amd64.tar.gz
sudo mv  velero-v1.3.2-linux-amd64 /usr/local/bin

# TODO : Add Velero to you PATH
PATH="/usr/local/bin/velero-v1.3.2-linux-amd64:$PATH"
# . ~/.profile
# ~/.bashrc 

# Command line Autocompletion : https://velero.io/docs/v1.3.2/customize-installation/#optional-velero-cli-configurations

source <(velero completion bash)
echo "source <(velero completion bash)" >> ~/.bashrc 
echo 'alias v=velero' >>~/.bashrc
echo 'complete -F __start_velero v' >>~/.bashrc

velero version

AZURE_BACKUP_SUBSCRIPTION_ID=$subId
SNAPSHOT_TIMEOUT=5m
AZURE_BACKUP_RESOURCE_GROUP="rg-backup-${appName}-${location}" 
az group create --name $AZURE_BACKUP_RESOURCE_GROUP --location $location

# https://github.com/vmware-tanzu/velero-plugin-for-microsoft-azure#setup
velero install \
    --provider azure \
    --plugins velero/velero-plugin-for-microsoft-azure:v1.0.1 \
    --bucket $BLOB_CONTAINER \
    --secret-file ./credentials-velero \
    --backup-location-config resourceGroup=$AZURE_BACKUP_RESOURCE_GROUP,storageAccount=$AZURE_STORAGE_ACCOUNT_ID,subscriptionId=$AZURE_BACKUP_SUBSCRIPTION_ID \
    --snapshot-location-config apiTimeout=$SNAPSHOT_TIMEOUT,resourceGroup=$AZURE_BACKUP_RESOURCE_GROUP,subscriptionId=$AZURE_BACKUP_SUBSCRIPTION_ID


# Install and configure the server components: https://vmware-tanzu.github.io/helm-charts/
velero_ns="velero"
k create namespace $velero_ns

# https://github.com/vmware-tanzu/helm-charts/blob/master/charts/velero/README.md
# https://github.com/vmware-tanzu/helm-charts/blob/master/charts/velero/values.yaml

# https://github.com/vmware-tanzu/velero-plugin-for-microsoft-azure/issues/34
# https://github.com/vmware-tanzu/helm-charts/blob/master/charts/velero/values.yaml#L96
# https://github.com/vmware-tanzu/velero-plugin-for-microsoft-azure/blob/v1.0.1/backupstoragelocation.md
helm install velero vmware-tanzu/velero --namespace $velero_ns \
--set configuration.provider=azure \
--set-file credentials.secretContents.cloud=./credentials-velero \
--set configuration.backupStorageLocation.name=azure \
--set configuration.backupStorageLocation.bucket=$BLOB_CONTAINER \
--set configuration.backupStorageLocation.config.region=$location \
--set configuration.volumeSnapshotLocation.name=azure \
--set configuration.volumeSnapshotLocation.config.region==$location \
--set image.repository=velero/velero \
--set image.pullPolicy=IfNotPresent \
--set initContainers[0].name=velero-plugin-for-microsoft-azure \
--set initContainers[0].image=velero/velero-plugin-for-microsoft-azure:v1.0.1 \
--set initContainers[0].volumeMounts[0].mountPath=/target \
--set initContainers[0].volumeMounts[0].name=plugins

helm ls -n $velero_ns -o yaml
helm status velero -n c

k get role -n $velero_ns | grep -i "velero"
k get rolebinding -n $velero_ns | grep -i "velero"
k get ClusterRole | grep -i "velero"
k get ClusterRoleBinding | grep -i "velero"
k get cm -n $velero_ns  | grep -i "velero"
k get sa -n $velero_ns | grep -i "velero"
k get svc -n $velero_ns
k get deploy -n  $velero_ns | grep -i "velero"
k get rs -n $velero_ns | grep -i "velero"
k get po -n $velero_ns -o wide | grep -i "velero"
k get crds -l app.kubernetes.io/name=velero -o wide

for pod in $(k get po -l "app.kubernetes.io/name=velero" -n $velero_ns -o=name)
do
  k describe $pod -n  $velero_ns # | grep -i "Error"
  k logs $pod $velero_ns #| grep -i "Error"
done


#for secret in $(k get secrets -n $velero_ns -o custom-columns=:metadata.name | grep -i "velero")
#do
#  k describe secret $secret -n $velero_ns
#  echo -e "\n"
#  velero_secret=$(k get secret $secret -n $velero_ns -o jsonpath="{.data.token}" | base64 --decode)
#  echo "Velero secret " 
#  echo -e "\n"
#  echo $velero_secret
#done


```

# Clean-Up
```sh

az storage container delete -n $BLOB_CONTAINER --account-name $AZURE_STORAGE_ACCOUNT_ID
az storage account delete --name $AZURE_STORAGE_ACCOUNT_ID -g $rg_name -y
az ad app delete --id velero-$appName

helm uninstall velero -n $velero_ns
helm ls -n $velero_ns -o yaml

# https://velero.io/docs/v1.3.2/uninstalling/

k delete namespace $velero_ns clusterrolebinding/velero
k delete crds -l component=velero

```