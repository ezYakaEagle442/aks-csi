# Setup Key-Vault

See also :
- [https://docs.microsoft.com/en-us/azure/aks/developer-best-practices-pod-security#use-pod-managed-identities](https://docs.microsoft.com/en-us/azure/aks/developer-best-practices-pod-security#use-pod-managed-identities)
- [https://docs.microsoft.com/en-us/azure/key-vault/key-vault-soft-delete-cli](https://docs.microsoft.com/en-us/azure/key-vault/key-vault-soft-delete-cli)
- [https://github.com/Azure/secrets-store-csi-driver-provider-azure](https://github.com/Azure/secrets-store-csi-driver-provider-azure)
- [https://docs.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/services-support-managed-identities#azure-key-vault](https://docs.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/services-support-managed-identities#azure-key-vault)

## Pre-requisites

```sh

az provider register -n Microsoft.KeyVault

# https://docs.microsoft.com/en-us/azure/key-vault/key-vault-soft-delete-cli
# az keyvault list-deleted
# az keyvault purge --name $vault_name --location $location

az keyvault create --name $vault_name --enable-soft-delete true --location $location -g $rg_name
az keyvault show --name $vault_name 
az keyvault update --name $vault_name --default-action deny -g $rg_name 

kv_id=$(az keyvault show --name $vault_name -g $rg_name --query "id" --output tsv)
echo "KeyVault ID :" $kv_id

nslookup $vault_name.vault.azure.net

```

## Update network rules
```sh
# https://docs.microsoft.com/en-us/azure/key-vault/general/network-security
az keyvault network-rule list --name $vault_name -g $rg_name
az network vnet list-endpoint-services -l $location

# Allow AKS cluster
az network vnet subnet update -g $rg_name --vnet-name $vnet_name --name $subnet_name --service-endpoints "Microsoft.KeyVault"
az keyvault network-rule add --name $vault_name -g $rg_name --subnet $subnet_id

```
