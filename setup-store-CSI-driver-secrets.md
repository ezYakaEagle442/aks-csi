
See also :
- [Limit credential exposure](https://docs.microsoft.com/en-us/azure/aks/developer-best-practices-pod-security#limit-credential-exposure)
- [Use Azure Key Vault with Secrets Store CSI Driver](https://docs.microsoft.com/en-us/azure/aks/developer-best-practices-pod-security#use-azure-key-vault-with-secrets-store-csi-driver)
- [https://github.com/Azure/secrets-store-csi-driver-provider-azure](https://github.com/Azure/secrets-store-csi-driver-provider-azure)
- [Pre-req](https://github.com/Azure/secrets-store-csi-driver-provider-azure#install-the-secrets-store-csi-driver-and-the-azure-keyvault-provider) : AKS 1.16+
CSI driver provides the ability to [sync with Kubernetes secrets, which can then be referenced by an environment variable.](https://github.com/kubernetes-sigs/secrets-store-csi-driver#optional-sync-with-kubernetes-secrets)



# Install the Secrets Store CSI Driver and the Azure Keyvault Provider

[Installing the Chart](https://github.com/Azure/secrets-store-csi-driver-provider-azure/blob/master/charts/csi-secrets-store-provider-azure/README.md)
```sh
helm install csi-secrets-store-provider-azure csi-secrets-store-provider-azure/csi-secrets-store-provider-azure -n $target_namespace
helm ls -n $target_namespace -o yaml
helm status csi-secrets-store-provider-azure -n $target_namespace
```

# Create key-Vault Secret
```sh

# https://docs.microsoft.com/en-us/cli/azure/keyvault/secret?view=azure-cli-latest#az-keyvault-secret-set
az keyvault secret set --name $vault_secret_name --value $vault_secret --description "CSI driver ${appName} Secret" --vault-name $vault_name
az keyvault secret list --vault-name $vault_name
az keyvault secret show --vault-name $vault_name --name $vault_secret_name --output tsv

az keyvault key create --name key1 --vault-name $vault_name --size 2048 --kty RSA
az keyvault key list --vault-name $vault_name
az keyvault key show --name key1 --vault-name $vault_name

aks_client_id=$(az aks show -g $rg_name -n $cluster_name --query identityProfile.kubeletidentity.clientId -o tsv)
echo "AKS Cluster Identity Client ID " $aks_client_id
az role assignment create --role Reader --assignee $aks_client_id --scope /subscriptions/$subId/resourcegroups/$rg_name/providers/Microsoft.KeyVault/vaults/$vault_name # $kv_id

```

## Perform role assignments

See [https://github.com/Azure/aad-pod-identity/blob/master/docs/readmes/README.role-assignment.md#performing-role-assignments](https://github.com/Azure/aad-pod-identity/blob/master/docs/readmes/README.role-assignment.md#performing-role-assignments)

```sh
# https://docs.microsoft.com/en-us/azure/role-based-access-control/built-in-roles#managed-identity-operator
# https://docs.microsoft.com/en-us/azure/role-based-access-control/built-in-roles#virtual-machine-contributor
az role assignment create --role "Managed Identity Operator" --assignee $aks_client_id --scope /subscriptions/$subId/resourcegroups/$managed_rg
az role assignment create --role "Virtual Machine Contributor" --assignee $aks_client_id --scope /subscriptions/$subId/resourcegroups/$managed_rg
```

## Install AAD Pod Identity

<span style="color:red">/!\ IMPORTANT </span> : For AKS clusters with limited egress-traffic, Please install pod-identity in kube-system namespace using the [helm charts](https://github.com/Azure/aad-pod-identity/tree/master/charts/aad-pod-identity).

```sh
helm install aad-pod-identity aad-pod-identity/aad-pod-identity -n $target_namespace --set azureIdentity.namespace=$target_namespace
helm ls -n $target_namespace -o yaml
helm status aad-pod-identity -n $target_namespace

# k apply -f https://raw.githubusercontent.com/Azure/aad-pod-identity/master/deploy/infra/deployment-rbac.yaml
k get crd -A
k api-resources
k describe ds aad-pod-identity-nmi -n $target_namespace 
k get deployments -n $target_namespace 
k get deployment aad-pod-identity-mic -n $target_namespace 
k describe deployment aad-pod-identity-mic -n $target_namespace 
k get pods -n $target_namespace 
for pod in $(k get pods -l app.kubernetes.io/component=mic -n $target_namespace -o custom-columns=:metadata.name)
do
	k describe pod $pod -n $target_namespace # | grep -i "Error"
	k logs $pod -n $target_namespace | grep -i "Error"
done

```

# Create an Azure User Identity


```sh
export SUBSCRIPTION_ID=$subId
export RESOURCE_GROUP=$rg_name
export IDENTITY_NAME="${appName}-csi-demo-pod-identity" #  must consist of lower case 

# https://github.com/Azure/aad-pod-identity/pull/48
# https://github.com/Azure/aad-pod-identity/issues/38
az identity create -g $rg_name -n $IDENTITY_NAME
export IDENTITY_CLIENT_ID="$(az identity show -g $rg_name -n $IDENTITY_NAME --query clientId -o tsv)"
export IDENTITY_RESOURCE_ID="$(az identity show -g $rg_name -n $IDENTITY_NAME --query id -o tsv)"
export IDENTITY_ASSIGNMENT_ID="$(az role assignment create --role Reader --assignee $IDENTITY_CLIENT_ID --scope /subscriptions/$subId/resourceGroups/$rg_name --query id -o tsv)"

# set policy to access keys in your keyvault
az keyvault set-policy -n $vault_name --key-permissions get --spn $IDENTITY_CLIENT_ID
# set policy to access secrets in your keyvault
az keyvault set-policy -n $vault_name --secret-permissions get --spn $IDENTITY_CLIENT_ID
# set policy to access certs in your keyvault
az keyvault set-policy -n $vault_name --certificate-permissions get --spn $IDENTITY_CLIENT_ID

tenantId=$(az account show --query tenantId -o tsv)

export TENANT_ID=$tenantId
export KV_NAME=$vault_name
export SECRET_NAME=$vault_secret_name
export POD_ID=xxx

envsubst < ./cnf/secrets-store-csi-provider-class.yaml > deploy/secrets-store-csi-provider-class.yaml
cat deploy/secrets-store-csi-provider-class.yaml
k apply -f deploy/secrets-store-csi-provider-class.yaml -n $target_namespace
k get secretproviderclasses -n $target_namespace
k describe secretproviderclasses azure-$KV_NAME-$POD_ID -n $target_namespace

export ResourceID=$IDENTITY_RESOURCE_ID
export ClientID=$IDENTITY_CLIENT_ID
envsubst < ./cnf/csi-demo-pod-identity.yaml > deploy/csi-demo-pod-identity.yaml
cat deploy/csi-demo-pod-identity.yaml

k apply -f deploy/csi-demo-pod-identity.yaml -n $target_namespace
k get azureidentity -A
k get azureidentitybindings -A
k get azureassignedidentities -A

k describe azureidentity $IDENTITY_NAME -n $target_namespace
k describe azureidentitybinding $IDENTITY_NAME-binding -n $target_namespace
k describe azureassignedidentities nginx-secrets-store-inline-$target_namespace-$IDENTITY_NAME -n $target_namespace

envsubst < ./cnf/secrets-store-csi-demo-pod.yaml > deploy/secrets-store-csi-demo-pod.yaml
cat deploy/secrets-store-csi-demo-pod.yaml
k apply -f deploy/secrets-store-csi-demo-pod.yaml -n $target_namespace
k get po -n $target_namespace -o wide
k get events -n $target_namespace | grep -i "Error" 
k describe pod nginx-secrets-store-inline -n $target_namespace
k get azureassignedidentities -A
k describe azureassignedidentities nginx-secrets-store-inline-$target_namespace-$IDENTITY_NAME -n $target_namespace
k logs nginx-secrets-store-inline -n $target_namespace

vmss_name=$(az vmss list -g $managed_rg --query [0].name -o tsv)
echo "VMSS name: " $vmss_name

node0_name=$(az vmss list-instances --name $vmss_name -g $managed_rg --query [0].name -o tsv)
echo "Node0 VM name: " $node0_name

az vmss identity show -g $managed_rg --name $vmss_name

```

# Test

```sh
k exec -it nginx-secrets-store-inline -n $target_namespace -- ls /mnt/secrets-store/ 
k exec -it nginx-secrets-store-inline -n $target_namespace -- cat /mnt/secrets-store/key1
k exec -it nginx-secrets-store-inline -n $target_namespace -- cat /mnt/secrets-store/$vault_secret_name

```