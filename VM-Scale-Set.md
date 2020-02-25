```bash
echo $RANDOM

export rg=azvmss8812
export vmss=azvmss8812

```

---

### Create cloud-init.txt

```yaml
#cloud-config
package_upgrade: true
packages:
  - nginx
  - nodejs
  - npm
write_files:
  - owner: www-data:www-data
  - path: /etc/nginx/sites-available/default
    content: |
      server {
        listen 80;
        location / {
          proxy_pass http://localhost:3000;
          proxy_http_version 1.1;
          proxy_set_header Upgrade $http_upgrade;
          proxy_set_header Connection keep-alive;
          proxy_set_header Host $host;
          proxy_cache_bypass $http_upgrade;
        }
      }
  - owner: azureuser:azureuser
  - path: /home/azureuser/myapp/index.js
    content: |
      var express = require('express')
      var app = express()
      var os = require('os');
      app.get('/', function (req, res) {
        res.send('Hello World from host ' + os.hostname() + '!')
      })
      app.listen(3000, function () {
        console.log('Hello world app listening on port 3000!')
      })
runcmd:
  - service nginx restart
  - cd "/home/azureuser/myapp"
  - npm init
  - npm install express -y
  - nodejs index.js

```

---

### Create VM Scale-set

```bash
az group create --name $rg --location southeastasia

az vmss create \
  --resource-group $rg \
  --name $vmss \
  --image UbuntuLTS \
  --upgrade-policy-mode automatic \
  --custom-data cloud-init.txt \
  --admin-username azureuser \
  --generate-ssh-keys

export vmsslb=$vmss"LB"
export vmsspubip=$vmss"LBPublicIP"
export vmssLBRuleWeb=$vmss"LoadBalancerRuleWeb"
export vmssLBBBEPool=$vmss"LBBEPool"


az network lb rule create \
  --resource-group $rg \
  --name $vmssLBRuleWeb \
  --lb-name $vmsslb \
  --backend-pool-name $vmssLBBBEPool \
  --backend-port 80 \
  --frontend-ip-name loadBalancerFrontEnd \
  --frontend-port 80 \
  --protocol tcp

az network public-ip show \
    --resource-group $rg \
    --name $vmsspubip \
    --output yaml

az vmss list-instances \
  --resource-group $rg \
  --name $vmss \
  --output table


az vmss list-instance-connection-info \
    --resource-group $rg \
    --name $vmss

```

### Increase Instance

```bash

az vmss show \
    --resource-group $rg \
    --name $vmss \
    --query [sku.capacity] \
    --output table

az vmss scale \
    --resource-group $rg \
    --name $vmss \
    --new-capacity 10

```

