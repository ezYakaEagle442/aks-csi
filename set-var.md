# Set-up environment variables

<span style="color:red">/!\ IMPORTANT </span> : your **appName** & **cluster_name** values MUST BE UNIQUE

## AKS Core variables
```sh

# az account list-locations : francecentral | northeurope | westeurope | eastus2
location=francecentral 
echo "location is : " $location 

appName="strcsi" 
echo "appName is : " $appName 

rg_name="rg-${appName}-${location}" 
echo "ARO RG name:" $rg_name 

cluster_name="aks-${appName}-101" #aro-<App Name>-<Environment>-<###>
echo "Cluster name:" $cluster_name

# AKS VNet & Subnet
vnet_name="vnet-${appName}"
echo "VNet Name :" $vnet_name

subnet_name="snet-${appName}"
echo "AKS Subnet Name :" $subnet_name

pod_cidr=10.42.0.0/24
echo "Pod CIDR is : " $pod_cidr 

svc_cidr=10.21.0.0/27
echo "service CIDR is : " $svc_cidr 

ssh_passphrase="<your secret>"
ssh_key="${appName}-key" # id_rsa

# Storage account name must be between 3 and 24 characters in length and use numbers and lower-case letters only

storage_name="stfr""${appName,,}"
echo "Storage name:" $storage_name

target_namespace="staging"
echo "Target namespace:" $target_namespace

vault_secret="NoSugarNoStar" 
echo "Vault secret:" $vault_secret 


```

## Extra variables
Note: The here under variables are built based on the varibales defined above, you should not need to modify them, just run this snippet

```sh
vault_name="kv-${appName}"
echo "Vault name :" $vault_name

vault_secret_name="${appName}-secret"
echo "Vault secret name:" $vault_secret_name 

node_pool_name="${appName}aronp"
echo "Node Pool name:" $node_pool_name
```