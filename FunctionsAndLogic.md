
### Creat Local Function

- FunctionName: TurbineRepair
- Authentication Level: Function

```csharp
using System;
using System.IO;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Mvc;
using Microsoft.Azure.WebJobs;
using Microsoft.Azure.WebJobs.Extensions.Http;
using Microsoft.AspNetCore.Http;
using Microsoft.Extensions.Logging;
using Newtonsoft.Json;


namespace Contoso.Function
{

    public static class TurbineRepair
    {
        const double revenuePerkW = 0.12;
        const double technicianCost = 250;
        const double turbineCost = 100;
    
        [FunctionName("TurbineRepair")]
        public static async Task<IActionResult> Run(
            [HttpTrigger(AuthorizationLevel.Function, "get", "post", Route = null)] HttpRequest req,
            ILogger log)
        {
               // Get query strings if they exist
            int tempVal;
            int? hours = Int32.TryParse(req.Query["hours"], out tempVal) ? tempVal : (int?)null;
            int? capacity = Int32.TryParse(req.Query["capacity"], out tempVal) ? tempVal : (int?)null;
        
            // Get request body
            string requestBody = await new StreamReader(req.Body).ReadToEndAsync();
            dynamic data = JsonConvert.DeserializeObject(requestBody);
        
            // Use request body if a query was not sent
            capacity = capacity ?? data?.capacity;
            hours = hours ?? data?.hours;
        
            // Return bad request if capacity or hours are not passed in
            if (capacity == null || hours == null){
                return new BadRequestObjectResult("Please pass capacity and hours on the query string or in the request body");
            }
            // Formulas to calculate revenue and cost
            double? revenueOpportunity = capacity * revenuePerkW * 24;  
            double? costToFix = (hours * technicianCost) +  turbineCost;
            string repairTurbine;
        
            if (revenueOpportunity > costToFix){
                repairTurbine = "Yes";
            }
            else {
                repairTurbine = "No";
            };
        
            return (ActionResult)new OkObjectResult(new{
                message = repairTurbine,
                revenueOpportunity = "$"+ revenueOpportunity,
                costToFix = "$"+ costToFix
            });
        }
    }
}
```


---
### Test Function
```bash
curl -v -H "Content-Type: application/json" -X POST --data-ascii '{"hours": "6","capacity": "2500"}' http://localhost:7071/api/TurbineRepair
```

---
### Publish to Azure

![href](https://docs.microsoft.com/en-us/azure/azure-functions/media/functions-openapi-definition/test-function.png)

```bash
curl -v -H "Content-Type: application/json" -X POST --data-ascii '{"hours": "6","capacity": "2500"}' https://smi15fxturbine.azurewebsites.net/api/TurbineRepair?code=9ZmXq4udbs1WkbNuhi7VmLpxU1Z2P8JFJs78ChjgAOcKzfiWVw0B5A==
```

![href](https://docs.microsoft.com/en-us/azure/azure-functions/media/functions-openapi-definition/select-all-settings-openapi.png)


---

### Put Message to Queue


```python
from azure.storage.queue import QueueService

queue_service = QueueService(account_name='smi15fxturbine', account_key='oo8uDXUOA5Bujq945QWnWZM0WTOZzPFlm21BMiN81CZwOW2UH6sfE66OpI8DSoPWXHKon04O9UG9bnH8xJrZNQ==')
queue_service.put_message('myqueue', u'{"hours": "6","capacity": "2500"}')
queue_service.put_message('myqueue', u'{"hours": "60","capacity": "2500"}')
queue_service.put_message('myqueue', u'{"hours": "600","capacity": "2500"}')
queue_service.put_message('myqueue', u'{"hours": "6000","capacity": "25000"}')
queue_service.put_message('myqueue', u'{"hours": "60000","capacity": "25000"}')
```

Message Payload
```json
{"message":"Yes","revenueOpportunity":"$7200","costToFix":"$1600"}
```


```json
{
    "definition": {
        "$schema": "https://schema.management.azure.com/providers/Microsoft.Logic/schemas/2016-06-01/workflowdefinition.json#",
        "actions": {
            "Condition": {
                "actions": {
                    "Set_variable": {
                        "inputs": {
                            "name": "requestStatus",
                            "value": "Approved"
                        },
                        "runAfter": {},
                        "type": "SetVariable"
                    }
                },
                "else": {
                    "actions": {
                        "Set_variable_2": {
                            "inputs": {
                                "name": "requestStatus",
                                "value": "Rejected"
                            },
                            "runAfter": {},
                            "type": "SetVariable"
                        }
                    }
                },
                "expression": {
                    "and": [
                        {
                            "equals": [
                                "@body('Parse_JSON')?['message']",
                                "Yes"
                            ]
                        }
                    ]
                },
                "runAfter": {
                    "Initialize_variable": [
                        "Succeeded"
                    ]
                },
                "type": "If"
            },
            "Delete_message": {
                "inputs": {
                    "host": {
                        "connection": {
                            "name": "@parameters('$connections')['azurequeues']['connectionId']"
                        }
                    },
                    "method": "delete",
                    "path": "/@{encodeURIComponent('myqueue')}/messages/@{encodeURIComponent(triggerBody()?['MessageId'])}",
                    "queries": {
                        "popreceipt": "@triggerBody()?['PopReceipt']"
                    }
                },
                "runAfter": {
                    "TurbineRepair": [
                        "Succeeded"
                    ]
                },
                "type": "ApiConnection"
            },
            "Initialize_variable": {
                "inputs": {
                    "variables": [
                        {
                            "name": "requestStatus",
                            "type": "string",
                            "value": "default"
                        }
                    ]
                },
                "runAfter": {
                    "Parse_JSON": [
                        "Succeeded"
                    ]
                },
                "type": "InitializeVariable"
            },
            "Parse_JSON": {
                "inputs": {
                    "content": "@body('TurbineRepair')",
                    "schema": {
                        "properties": {
                            "costToFix": {
                                "type": "string"
                            },
                            "message": {
                                "type": "string"
                            },
                            "revenueOpportunity": {
                                "type": "string"
                            }
                        },
                        "type": "object"
                    }
                },
                "runAfter": {
                    "Delete_message": [
                        "Succeeded"
                    ]
                },
                "type": "ParseJson"
            },
            "Send_an_email_(V2)_3": {
                "inputs": {
                    "body": {
                        "Body": "<p>Request:<br>\n@{triggerBody()?['MessageText']}<br>\n<br>\nResult:<br>\n@{body('TurbineRepair')}</p>",
                        "Subject": "Turbine repair request : @{variables('requestStatus')}: @{triggerBody()?['MessageId']}",
                        "To": "smithm@microsoft.com"
                    },
                    "host": {
                        "connection": {
                            "name": "@parameters('$connections')['office365']['connectionId']"
                        }
                    },
                    "method": "post",
                    "path": "/v2/Mail"
                },
                "runAfter": {
                    "Condition": [
                        "Succeeded"
                    ]
                },
                "type": "ApiConnection"
            },
            "TurbineRepair": {
                "inputs": {
                    "body": "@triggerBody()?['MessageText']",
                    "function": {
                        "id": "/subscriptions/a5690002-a0db-48a6-bd96-634ce65c45ee/resourceGroups/smi15fxturbine/providers/Microsoft.Web/sites/smi15fxturbine/functions/TurbineRepair"
                    }
                },
                "runAfter": {},
                "type": "Function"
            }
        },
        "contentVersion": "1.0.0.0",
        "outputs": {},
        "parameters": {
            "$connections": {
                "defaultValue": {},
                "type": "Object"
            }
        },
        "triggers": {
            "When_there_are_messages_in_a_queue": {
                "inputs": {
                    "host": {
                        "connection": {
                            "name": "@parameters('$connections')['azurequeues']['connectionId']"
                        }
                    },
                    "method": "get",
                    "path": "/@{encodeURIComponent('myqueue')}/message_trigger"
                },
                "recurrence": {
                    "frequency": "Minute",
                    "interval": 3
                },
                "splitOn": "@triggerBody()?['QueueMessagesList']?['QueueMessage']",
                "type": "ApiConnection"
            }
        }
    },
    "parameters": {
        "$connections": {
            "value": {
                "azurequeues": {
                    "connectionId": "/subscriptions/a5690002-a0db-48a6-bd96-634ce65c45ee/resourceGroups/smi15fxturbine/providers/Microsoft.Web/connections/azurequeues",
                    "connectionName": "azurequeues",
                    "id": "/subscriptions/a5690002-a0db-48a6-bd96-634ce65c45ee/providers/Microsoft.Web/locations/southeastasia/managedApis/azurequeues"
                },
                "office365": {
                    "connectionId": "/subscriptions/a5690002-a0db-48a6-bd96-634ce65c45ee/resourceGroups/smi15fxturbine/providers/Microsoft.Web/connections/office365",
                    "connectionName": "office365",
                    "id": "/subscriptions/a5690002-a0db-48a6-bd96-634ce65c45ee/providers/Microsoft.Web/locations/southeastasia/managedApis/office365"
                }
            }
        }
    }
}
```