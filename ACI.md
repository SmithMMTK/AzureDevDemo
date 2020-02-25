```bash
echo $RANDOM

export rg=az-aci-17295
export aci=az-myaci-17295
export dnslabel=az-myaci-17295


az group create --name $rg --location southeastasia

az container create --resource-group $rg --name $aci --image nginxdemos/hello --dns-name-label $dnslabel --ports 80
#az container create --resource-group $rg --name $aci --image mcr.microsoft.com/azuredocs/aci-helloworld --dns-name-label $dnslabel --ports 80

az container show --resource-group $rg --name $aci --query "{FQDN:ipAddress.fqdn,ProvisioningState:provisioningState}" --out table


```