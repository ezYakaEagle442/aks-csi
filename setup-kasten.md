# Setup Kasten

See :
- [https://docs.kasten.io/install/azure/azure.html](https://docs.kasten.io/install/azure/azure.html)
- []()



```sh

oc create namespace kasten-io

helm install k10 kasten/k10 --namespace=kasten-io \
    --set secrets.azureTenantId=<tenantID> \
    --set secrets.azureClientId=<azureclient_id> \
    --set secrets.azureClientSecret=<azureclientsecret>
    --set persistence.storageClass=<storage-class-name> \
    --set prometheus.server.persistentVolume.storageClass=<storage-class-name>
    --set scc.create=true # ARO

k get pods --namespace kasten-io --watch
k --namespace kasten-io port-forward service/gateway 8080:8000
# The K10 dashboard will be available at http://127.0.0.1:8080/k10/#/.



```