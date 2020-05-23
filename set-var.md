# Set-up environment variables

<span style="color:red">/!\ IMPORTANT </span> : your **appName** & **cluster_name** values MUST BE UNIQUE

## AKS Core variables
```sh

# Target version : 1.17.3 to leverage PV Snapshot & Volume Limits
version=$(az aks get-versions -l $location --query 'orchestrators[-3].orchestratorVersion' -o tsv) 
echo "version is :" $version 

appName="strcsi" 
echo "appName is : " $appName 

# az account list-locations : francecentral | northeurope | westeurope | eastus2
location=francecentral 
echo "location is : " $location 

# Storage account name must be between 3 and 24 characters in length and use numbers and lower-case letters only
storage_name="stfr""${appName,,}"
echo "Storage name:" $storage_name

rg_name="rg-${appName}-${location}" 
echo "ARO RG name:" $rg_name 

cluster_name="aks-${appName}-101" #aks-<App Name>-<Environment>-<###>
echo "Cluster name:" $cluster_name

network_plugin="azure"
echo "Network Plugin is : " $network_plugin 

network_policy="azure"
echo "Network Policy is : " $network_policy 

# AKS VNet & Subnet
vnet_name="vnet-${appName}"
echo "VNet Name :" $vnet_name

subnet_name="snet-${appName}"
echo "AKS Subnet Name :" $subnet_name

svc_cidr=10.42.0.0/24
echo "service CIDR is : " $svc_cidr 

target_namespace="staging"
echo "Target namespace:" $target_namespace

vault_secret="NoSugarNoStar" 
echo "Vault secret:" $vault_secret 

ssh_passphrase="<your secret>"
ssh_key="${appName}-key" # id_rsa

vault_name="kv-${appName}"
echo "Vault name :" $vault_name

aks_admin_username="${appName}-admin"
echo "AKS admin user-name :" $aks_admin_username

vault_secret_name="${appName}-secret"
echo "Vault secret name:" $vault_secret_name 

node_pool_name="${appName}aksnp"
echo "Node Pool name:" $node_pool_name
```