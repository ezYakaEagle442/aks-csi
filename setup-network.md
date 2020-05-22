## Plan IP addressing for your cluster

See  :

- [Public IP sku comparison](https://docs.microsoft.com/en-us/azure/virtual-network/virtual-network-ip-addresses-overview-arm#sku)

- [https://docs.microsoft.com/en-us/azure/aks/configure-azure-cni#plan-ip-addressing-for-your-cluster](https://docs.microsoft.com/en-us/azure/aks/configure-azure-cni#plan-ip-addressing-for-your-cluster)

- For Advanced networking options, see [https://docs.microsoft.com/en-us/azure/aks/configure-azure-cni](https://docs.microsoft.com/en-us/azure/aks/configure-azure-cni)

```sh
# https://www.ipaddressguide.com/cidr
# https://github.com/Azure/azure-quickstart-templates/tree/master/101-aks-advanced-networking-aad
# https://github.com/Azure/azure-quickstart-templates/tree/master/101-aks-advanced-networking

``` 

## Create Networks

```sh
# AKS nodes VNet & Subnet
az network vnet create --name $vnet_name --resource-group $rg_name --address-prefixes 172.16.0.0/16 --location $location
az network vnet subnet create --name $subnet_name --address-prefixes 172.16.1.0/24 --vnet-name $vnet_name --resource-group $rg_name 

vnet_id=$(az network vnet show --resource-group $rg_name --name $vnet_name --query id -o tsv)
echo "VNet Id :" $vnet_id	

subnet_id=$(az network vnet subnet show --resource-group $rg_name --vnet-name $vnet_name --name $subnet_name --query id -o tsv)
echo "Subnet Id :" $subnet_id	

```
