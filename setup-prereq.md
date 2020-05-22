# Create RG
```sh
az group create --name $rg_name --location $location
# az group create --name rg-cloudshell-$location --location $location

```

# Create Storage

This is not mandatory, you can create a storage account to play with CloudShell

```sh
# https://docs.microsoft.com/en-us/cli/azure/storage/account?view=azure-cli-latest#az-storage-account-create
# https://docs.microsoft.com/en-us/azure/storage/common/storage-introduction#types-of-storage-accounts
az storage account create --name stcloudshellfr --kind StorageV2 --sku Standard_LRS -g rg-cloudshell-$location --location $location --https-only true
# az storage account create --name $storage_name --kind StorageV2 --sku Standard_LRS --resource-group $rg_name --location $location --https-only true

```


# Generates your SSH keys

<span style="color:red">/!\ IMPORTANT </span> :  check & save your ssh_passphrase !!!

Generate & save nodes [SSH keys](https://docs.microsoft.com/en-us/azure/aks/ssh) to Azure Key-Vault is a [Best-practice](https://github.com/Azure/k8s-best-practices/blob/master/Security_securing_a_cluster.md#securing-host-access)

If you want to save your keys to keyVault, [KV must be created first](setup-kv.md)

Read [this explanation about private keys management in KV](https://github.com/Azure/azure-sdk-for-js/issues/7647#issuecomment-594935307)
The KeyVault service stores both the public and the private parts of your certificate in a KeyVault secret, along with any other secret you might have created in that same KeyVault instance. With this separation comes considerable control of the level of access you can give to people in your organization regarding your certificates. The access control can be specified through the policy you pass in when creating a certificate. Knowing that the private key is stored in a KeyVault Secret, with the public certificate included, we can retrieve it by using the KeyVault-Secrets client.

See also :
- [About keys](https://docs.microsoft.com/en-us/azure/key-vault/certificates/about-certificates#composition-of-a-certificate)
- Composition of a Certificate](https://docs.microsoft.com/en-us/azure/key-vault/certificates/about-certificates#composition-of-a-certificate)
- [https://github.com/MicrosoftDocs/azure-docs/issues/55072](https://github.com/MicrosoftDocs/azure-docs/issues/55072)
- [https://github.com/Azure/azure-cli/issues/13547](https://github.com/Azure/azure-cli/issues/13547)
- [https://github.com/Azure/azure-cli/issues/13548](https://github.com/Azure/azure-cli/issues/13548)

```sh
ssh-keygen -t rsa -b 4096 -N $ssh_passphrase -f ~/.ssh/$ssh_key -C "youremail@groland.grd"

# https://www.ssh.com/ssh/keygen/
# -y Read a private OpenSSH format file and print an OpenSSH public key to stdout.
# ssh-keygen -y -f ~/.ssh/id_rsa > ~/.ssh/id_rsa.pub
# ssh-keygen -l -f ~/.ssh/id_rsa
# az keyvault key create --name $ssh_key --vault-name $vault_name --size 2048 --kty RSA

az keyvault update --name $vault_name --default-action allow -g $rg_name 

az keyvault key import --name $ssh_key --vault-name $vault_name --pem-file ~/.ssh/$ssh_key --pem-password $ssh_passphrase
az keyvault key list --vault-name $vault_name
az keyvault key show --name $ssh_key --vault-name $vault_name
az keyvault key download --name $ssh_key --vault-name $vault_name --encoding PEM --file key2

az keyvault update --name $vault_name --default-action deny -g $rg_name 

```

KV stores both public part and private part of your key. Once you uploaded your key into KV, there is no way to show or download the private part: this is for security concern.
Once the key is imported in KV, then az keyvault key download returns the public key only. 

You can eventually save the private key key and also the passphrase as secrets in KV

See :
- this similar [topic & explanation regarding certificates](https://github.com/Azure/azure-sdk-for-js/issues/7647#issuecomment-594935307).
- [https://github.com/MicrosoftDocs/azure-docs/issues/55072](https://github.com/MicrosoftDocs/azure-docs/issues/55072)

```sh
az keyvault secret list --vault-name $vault_name
az keyvault secret show --name $vault_secret_name --vault-name $vault_name --output tsv

ssh_prv_key_val=`cat ~/.ssh/$ssh_key`

az keyvault secret set --name ssh-passphrase --value $ssh_passphrase --vault-name $vault_name --description "AKS ${appName} SSH Key Passphrase" 
az keyvault secret set --name ssh-key --value "$ssh_prv_key_val" --vault-name $vault_name --description "AKS ${appName} SSH Private Key value" 

az keyvault secret list --vault-name $vault_name
az keyvault secret show --vault-name $vault_name --name ssh-passphrase -o table
az keyvault secret show --vault-name $vault_name --name ssh-key -o table

az keyvault secret download --file myLostKey.txt --name ssh-key --vault-name $vault_name
az keyvault secret download --file myLostPassPhrase.txt --name ssh-passphrase --vault-name $vault_name
```