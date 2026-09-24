# The Friday Deploy

## Context and Scope
* A new service deployed by an intern late Friday into a lab resource group, reviewed before promotion to production.
* Live multi-user Azure training tenant, Reader access 



## Method
Compared the new deployment against an existing healthy production service in the shared resource group, across five review areas. There was no written standard and standard was derived from the working system.

## Findings

### THE WHERE
I observed the new deployment resource group was deployed in a completely different region from the production resource group. I verified this by opening up the Overview Blade for each resource group and saw that they were set to different regions for the location. This matters because Azure doesn't charge for any data you upload to it (Ingress), but will charge for any data leaving Azure services (Egress). In the current sitation if the new deployment were to call for any data from production, there will be latency for the data arriving in the new deployment, while the data leaving production would incur fees for the data egress. More data fees will occur if any data leaves the new deployment and is sent to production, creating a horrible combination of subpar performance with latency while racking completely unnesscessary fees. The new deployment should be reployed on the same Azure region to avoid the unnesscessary fees.

![Image showing prod in Central US]()
![Image showing new deployment not in Central US]()

### THE WHO
I observed in the new deployment the application had no managed identity attached, which means it authenticates with a stored credential. I verified this by opening the Identity blade of the application and looking at the User assigned tab and finding no user identities were found. I compared this to production's app, where it did have a user identity assigned. We want to use System or User assigned identities in Azure to store credentails, because the alternative is hardcoding the credentials into the code. This should not be done unless absolutely necessary, as having secrets directly in your code has a high potential of the hardcoded secrets being leaked. Code uploaded to Github or network activity traced are two easy ways for hardcoded secrets to be leaked. A managed identity should be assigned for the new deployment application remove this unnesscessary risk. Managed identities utilize either the Microsoft Authentication Library (MSAL) or Azure.Indentity SDK to retrieve a managed identity token from Entra ID backed by the managed identity. This token acquisition doesn't require any secrets and is automatically authenticated based on the environment where the code runs. 
![Prod showing User Assigned Identity]() 
![New Deployment having no User assigned Identities]()

### THE LEAK
I observed a storage container was configured for anonymous access. This means that the storage container is publicly availible on the internet and can be accessed by absolutely anyone and they don't even need to be authenticated to view the data. I verified the anonymous access by opening the blob URL in a private browser window and being able to see the contents within. 
![Image showing that container is enabled for anonymous access]()
![Image showing that container is viewable without being authenticated]()

### THE DOOR
I observed the function app in the new deployment had zero inbound access restrictions. I compared this to the current production function app and saw that production does have access restrictions in place; one all-encompassing deny rule.
![Image showing public network access enalbed with access restrictions ]() 
![Image showing public network access enalbed without access restrictions]()

### THE CALL
I determined the priority of applying fixes to these findings by applying this rule: Exposure beats Hygiene. Active beats Potential. The first thing to fix is the container with anonymous access enabled. The sensitive date within the container is completely exposed to the public internet and can be viewed by anyone. This is an event already in progress. The rest of the findings could have service tickets put in and be corrected after the priority exposure is corrected. This should be rectified as soon as humanly possible. 

## Recomendations
Close the public container immediately. Attach a user-assigned managed identity with a scoped role on the storage account and remove the stored credential. Redeploy into the platform's standard region. Add a single deny rule to close inbound, since one rule flips everything unmatched to implicit deny.