# Create AKS Cluster


```sh

az aks create --name $cluster_name \
    --resource-group $rg_name \
    --aks-custom-headers CustomizedUbuntu=aks-ubuntu-1804 \
    --node-count=1 \
    --node-vm-size Standard_F2s_v2 \
    --location $location \
    --vnet-subnet-id $subnet_id \
    --service-cidr 10.42.0.0/24 \
    --dns-service-ip 10.42.0.10 \
    --kubernetes-version $version \
    --network-plugin $network_plugin \
    --network-policy $network_policy \
    --nodepool-name $node_pool_name \
    --load-balancer-sku standard \
    --vm-set-type VirtualMachineScaleSets \
    --admin-username $aks_admin_username \
    --ssh-key-value ~/.ssh/${ssh_key}.pub \
    --enable-managed-identity \
    --verbose

```

## Get AKS Credentials

Apply [KubeCtl alias](./tools#kube-tools)

```sh

ls -al ~/.kube
rm  ~/.kube/config

az aks get-credentials --resource-group $rg_name --name $cluster_name --admin
az aks show -n $cluster_name -g $rg_name

aks_api_server_url=$(az aks show -n $cluster_name -g $rg_name --query 'Fqdn' -o tsv)
echo "AKS API server URL: " $aks_api_server_url

```


## Connect to the Cluster

```sh
az aks list -o table
kubectl cluster-info

```

## Create Namespaces
```sh
kubectl create namespace development
kubectl label namespace/development purpose=development

kubectl create namespace staging
kubectl label namespace/staging purpose=staging

kubectl create namespace production
kubectl label namespace/production purpose=production

kubectl create namespace sre
kubectl label namespace/sre purpose=sre

kubectl get namespaces
kubectl describe namespace production
kubectl describe namespace sre
```
