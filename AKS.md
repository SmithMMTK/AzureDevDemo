
```bash
echo $RANDOM
export rg=az-aks-29756
export aks=az-myAKSCluster-29756
export acr=smi15
export acrserver=smi15.azurecr.io

```

### Get application code
```bash
#git clone https://github.com/Azure-Samples/azure-voting-app-redis.git
```

### Create container images
```bash
docker-compose up -d
docker images
docker ps
open http://localhost:8080

# Stop and remove the container instances 
docker-compose down
```

### Create Azure Container Registry
```bash
az acr login --name $acr
az acr list --resource-group azCommon --query "[].{acrLoginServer:loginServer}" --output table

#docker image rm $acrserver/azure-vote-front:CloudReady -f
docker tag azure-vote-front $acrserver/azure-vote-front:CloudReady

```

### Push image to registry
```bash
docker push $acrserver/azure-vote-front:CloudReady
az acr repository list --name $acr --output table
az acr repository show-tags --name $acr --repository azure-vote-front --output table
```

### Create Azure Kubernestes Cluster
```bash
az group create -n $rg -l southeastasia
az aks create \
    --resource-group $rg \
    --name $aks \
    --node-count 2 \
    --generate-ssh-keys \
    --attach-acr $acr
```

### Connect to cluster using kubectl

```bash
#az aks install-cli 

az aks get-credentials --resource-group $rg --name $aks

#kubectl config get-contexts
#kubectl cluster-info
#kCONTEXT_NAME
#kubectl config current-context
#
#export targetd=az-aks-10407_az-myAKSCluster-10407
#
#kubectl config unset contexts.$targetd
#kubectl config unset users.clusterUser_$targetd
#kubectl config unset clusters.$targetd
#
# 

kubectl get nodes
kubectl config view

```

---

### Update the manifest file
```bash
### smi15.azurecr.io

code azure-vote-all-in-one-redis.yaml

```

### Deploy application
```bash
kubectl apply -f azure-vote-all-in-one-redis-local.yaml

kubectl get service azure-vote-front --watch
kubectl get all
kubectl get pods
kubectl scale --replicas=5 deployment/azure-vote-front

kubectl get pod azure-vote-front-xxx-yyy -o yaml 
kubectl get pod azure-vote-front-xxx-yyy -o yaml --export



```

---


