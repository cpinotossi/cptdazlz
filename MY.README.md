# ALZ


![High Level Deployment Flow](docs/wiki/media/high-level-deployment-flow.png)

## Deploy Management Groups Module

Based on

Modify the parameters file to suit your needs.
~~~bash
code infra-as-code/bicep/modules/managementGroups/parameters/managementGroups.parameters.all.json
# top parent management group id via azure cli
az account management-group list --query "[?displayName=='myedge.org'].id" -o tsv
# review the bicep file
code infra-as-code/bicep/modules/managementGroups/managementGroups.bicep
~~~

~~~bash
# instructions
code /home/azureuser/ALZ-Bicep/infra-as-code/bicep/modules/managementGroups/README.md
# For Azure global regions
dateYMD=$(date +%Y%m%dT%H%M%S%NZ)
NAME="alz-MGDeployment-${dateYMD}"
LOCATION="germanywestcentral"
TEMPLATEFILE="infra-as-code/bicep/modules/managementGroups/managementGroups.bicep"
PARAMETERS="@infra-as-code/bicep/modules/managementGroups/parameters/managementGroups.parameters.all.json"
az deployment tenant create --name ${NAME:0:63} --location $LOCATION --template-file $TEMPLATEFILE --parameters $PARAMETERS
~~~

## Deploy Custom Policy Definitions Module

~~~bash
# instructions
code /home/azureuser/ALZ-Bicep/infra-as-code/bicep/modules/policy/definitions/README.md
code infra-as-code/bicep/modules/policy/definitions/customPolicyDefinitions.bicep
# For Azure global regions
dateYMD=$(date +%Y%m%dT%H%M%S%NZ)
NAME="alz-PolicyDefsDefaults-${dateYMD}"
LOCATION="germanywestcentral"
MGID="alz"
TEMPLATEFILE="infra-as-code/bicep/modules/policy/definitions/customPolicyDefinitions.bicep"
PARAMETERS="@infra-as-code/bicep/modules/policy/definitions/parameters/customPolicyDefinitions.parameters.all.json"

az deployment mg create --name ${NAME:0:63} --location $LOCATION --management-group-id $MGID --template-file $TEMPLATEFILE --parameters $PARAMETERS
~~~

## Deploy Custom Policy Assignments Module

After you have deployed the policy module to add all of the custom ALZ Azure Policy Definitions & Initiatives you will need to assign the modules to the relevant Management Groups as per your requirements using the [Policy Assignments module](./infra-as-code/bicep/modules/policy/assignments/README.md).

If you want to make all of the default Azure Policy Assignments that we recommend in the Azure Landing Zones conceptual architecture and reference implementation you can use the [ALZ Default Policy Assignments module](./infra-as-code/bicep/modules/policy/assignments/alzDefaults/README.md) to do this for you👍

~~~bash
code /home/azureuser/ALZ-Bicep/infra-as-code/bicep/modules/policy/assignments/README.md
# copy the deny example
cp infra-as-code/bicep/modules/policy/assignments/parameters/policyAssignmentManagementGroup.deny.parameters.min.json infra-as-code/bicep/modules/policy/assignments/parameters/policyAssignmentManagementGroup.deny.pubip.parameters.min.json
code infra-as-code/bicep/modules/policy/assignments/parameters/policyAssignmentManagementGroup.deny.pubip.parameters.min.json

dateYMD=$(date +%Y%m%dT%H%M%S%NZ)
NAME="alz-alz-PolicyDenyAssignmentsDeployment-${dateYMD}"

PARAMETERS="@infra-as-code/bicep/modules/policy/assignments/parameters/policyAssignmentManagementGroup.deny.pubip.parameters.min.json"
LOCATION="germanywestcentral"
MGID="alz-landingzones"
TEMPLATEFILE="infra-as-code/bicep/modules/policy/assignments/policyAssignmentManagementGroup.bicep"

az deployment mg create --name ${NAME:0:63} --location $LOCATION --management-group-id $MGID --template-file $TEMPLATEFILE --parameters $PARAMETERS
~~~

If you wish to add your own additional custom Azure Policy Definitions please review [How Does ALZ-Bicep Implement Azure Policies?](https://github.com/Azure/ALZ-Bicep/wiki/PolicyDeepDive) and more specifically [Assigning Azure Policies](https://github.com/Azure/ALZ-Bicep/wiki/AssigningPolicies)


## Deploy custom RBAC Role definition

~~~bash
# instructions
code ./infra-as-code/bicep/modules/customRoleDefinitions/README.md
code infra-as-code/bicep/modules/customRoleDefinitions/customRoleDefinitions.bicep
# For Azure global regions

# Management Group ID
MGID="alz"

# Chosen Azure Region
LOCATION="germanywestcentral"

dateYMD=$(date +%Y%m%dT%H%M%S%NZ)
NAME="alz-CustomRoleDefsDeployment-${dateYMD}"
TEMPLATEFILE="infra-as-code/bicep/modules/customRoleDefinitions/customRoleDefinitions.bicep"
PARAMETERS="@infra-as-code/bicep/modules/customRoleDefinitions/parameters/customRoleDefinitions.parameters.all.json"

az deployment mg create --name ${NAME:0:63} --location $LOCATION --management-group-id $MGID --template-file $TEMPLATEFILE --parameters $PARAMETERS
~~~

## Deploy Logging & Sentinel Module

~~~bash
# instructions
code ./infra-as-code/bicep/modules/logging/README.md
code ./infra-as-code/bicep/modules/logging/parameters/logging.parameters.all.json
code ./infra-as-code/bicep/modules/logging/logging.bicep
# For Azure Global regions
# Set Platform management subscripion ID as the the current subscription
# your platform management subscription ID]
ManagementSubscriptionId=$(az account list --query "[?name=='sub-myedge-01'].id" -o tsv)
az account set --subscription $ManagementSubscriptionId

# Set the top level MG Prefix in accordance to your environment. This example assumes default 'alz'.
TopLevelMGPrefix="alz"

dateYMD=$(date +%Y%m%dT%H%M%S%NZ)
GROUP="rg-$TopLevelMGPrefix-logging-001"
NAME="alz-loggingDeployment-${dateYMD}"
LOCATION="germanywestcentral"
# Make sure to replace the location.
TEMPLATEFILE="infra-as-code/bicep/modules/logging/logging.bicep"
PARAMETERS="@infra-as-code/bicep/modules/logging/parameters/logging.parameters.all.json"

# Create Resource Group - optional when using an existing resource group
az group create --name $GROUP --location $LOCATION

# Deploy Module
az deployment group create --name ${NAME:0:63} --resource-group $GROUP --template-file $TEMPLATEFILE --parameters $PARAMETERS
# In case you like to overwrite the Locations
 --parameters parAutomationAccountLocation=$LOCATION parLogAnalyticsWorkspaceLocation=$LOCATION
~~~

Microsoft.OperationsManagement/solutions is an Azure resource provider used for deploying and managing Operations Management Suite (OMS) solutions. OMS is a collection of management services that were designed in the cloud from the start. Rather than deploying and managing on-premises resources, OMS components are entirely hosted in Azure.

OMS solutions are a collection of logic, visualization, and data acquisition rules that provide a consolidated view across your environment. Each solution might include logs, queries, and visualizations that provide insight into a particular area.

For example, you might use the Microsoft.OperationsManagement/solutions resource provider to deploy a Log Analytics solution, which can collect and analyze data generated by resources in your cloud and on-premises environments.

Please note that OMS has been deprecated and its functionality has been incorporated into Azure Monitor and Azure Security Center. However, the Microsoft.OperationsManagement/solutions resource provider is still used for managing these solutions.

## Deploy Management Groups Diagnostic Settings Module

~~~bash
# instructions
code ./infra-as-code/bicep/modules/mgDiagSettings/README.md
code ./infra-as-code/bicep/orchestration/mgDiagSettingsAll/README.md
code ./infra-as-code/bicep/orchestration/mgDiagSettingsAll/parameters/mgDiagSettingsAll.parameters.all.json
code ./infra-as-code/bicep/orchestration/mgDiagSettingsAll/mgDiagSettingsAll.bicep
# list all log analytics workspaces in germany west central
az monitor log-analytics workspace list --query "[?location=='germanywestcentral'].id" -o table
lawid=$(az monitor log-analytics workspace show -n alz-log-analytics -g rg-alz-logging-001 --query id -o tsv)

# For Azure global regions
# In this example, the Diagnostic Settings are enabled on the management groups through a managementGroup-scoped deployment.
TEMPLATEFILE="infra-as-code/bicep/orchestration/mgDiagSettingsAll/mgDiagSettingsAll.bicep"
PARAMETERS="@infra-as-code/bicep/orchestration/mgDiagSettingsAll/parameters/mgDiagSettingsAll.parameters.all.json"
LOCATION="germanywestcentral"
# Management Group ID
MGID="alz"
az deployment mg create --template-file $TEMPLATEFILE --parameters $PARAMETERS --location $LOCATION --management-group-id $MGID --parameters parLogAnalyticsWorkspaceResourceId=$lawid

diagSettingsName="toLa"
az rest -m GET --url https://management.azure.com/providers/microsoft.management/managementGroups/$MGID/providers/microsoft.insights/diagnosticSettings/$toLa --url-parameters api-version=2020-01-01-preview
az rest -h
~~~

~~~json
{
  "value": [
    {
      "id": "providers/Microsoft.Management/managementGroups/alz/providers/microsoft.insights/diagnosticSettings/toLa",
      "location": "global",
      "name": "toLa",
      "properties": {
        "logs": [
          {
            "category": "Administrative",
            "categoryGroup": null,
            "enabled": true
          },
          {
            "category": "Policy",
            "categoryGroup": null,
            "enabled": true
          }
        ],
        "workspaceId": "/subscriptions/f474dec9-5bab-47a3-b4d3-e641dac87ddb/resourceGroups/rg-alz-logging-001/providers/Microsoft.OperationalInsights/workspaces/alz-log-analytics"
      },
      "type": "Microsoft.Insights/diagnosticSettings"
    }
  ]
}
~~~

~~~bash
diagSettingsName="toLa"
MGIDPLATTFORM="alz-platform"
az rest -m GET --url https://management.azure.com/providers/microsoft.management/managementGroups/$MGIDPLATTFORM/providers/microsoft.insights/diagnosticSettings/$toLa --url-parameters api-version=2020-01-01-preview
az rest -h
~~~

~~~json
{
  "value": [
    {
      "id": "providers/Microsoft.Management/managementGroups/alz-platform/providers/microsoft.insights/diagnosticSettings/toLa",
      "location": "global",
      "name": "toLa",
      "properties": {
        "logs": [
          {
            "category": "Administrative",
            "categoryGroup": null,
            "enabled": true
          },
          {
            "category": "Policy",
            "categoryGroup": null,
            "enabled": true
          }
        ],
        "workspaceId": "/subscriptions/f474dec9-5bab-47a3-b4d3-e641dac87ddb/resourceGroups/rg-alz-logging-001/providers/Microsoft.OperationalInsights/workspaces/alz-log-analytics"
      },
      "type": "Microsoft.Insights/diagnosticSettings"
    }
  ]
}
~~~


## Deploy vWAN Network Module

~~~bash
# instructions
code ./infra-as-code/bicep/modules/hubNetworking/README.md
code ./infra-as-code/bicep/modules/vwanConnectivity/README.md
code ./infra-as-code/bicep/modules/hubNetworking/parameters/hubNetworking.parameters.all.json

# For Azure global regions

# Set Platform connectivity subscription ID as the the current subscription
ConnectivitySubscriptionId=$(az account list --query "[?name=='sub-myedge-01'].id" -o tsv)

az account set --subscription $ConnectivitySubscriptionId

# Set the top level MG Prefix in accordance to your environment. This example assumes default 'alz'.
TopLevelMGPrefix="alz"

dateYMD=$(date +%Y%m%dT%H%M%S%NZ)
NAME="alz-HubNetworkingDeploy-${dateYMD}"
GROUP="rg-$TopLevelMGPrefix-hub-networking-001"
TEMPLATEFILE="infra-as-code/bicep/modules/hubNetworking/hubNetworking.bicep"
PARAMETERS="@infra-as-code/bicep/modules/hubNetworking/parameters/hubNetworking.parameters.all.json"
LOCATION="germanywestcentral"
MGID="alz"

az group create --location eastus \
   --name $GROUP

az deployment group create --name ${NAME:0:63} --resource-group $GROUP --template-file $TEMPLATEFILE --parameters $PARAMETERS --parameters parLocation=$LOCATION parHubNetworkName=$MGID-hub-$LOCATION parHubNetworkAddressPrefix="10.0.0.0/16"
~~~

## Enterprise Azure Policy as Code (EPAC)

https://aka.ms/epac

based on https://azure.github.io/enterprise-azure-policy-as-code/quick-start/

~~~powershell
Install-Module Az -Scope CurrentUser
Connect-AzAccount -UseDeviceAuthentication
# show account details
Get-AzContext
# set subscription
Set-AzContext -SubscriptionId "f474dec9-5bab-47a3-b4d3-e641dac87ddb"
Install-Module EnterprisePolicyAsCode -Scope CurrentUser
New-EPACDefinitionFolder -DefinitionsRootFolder Definitions
~~~

## Git

~~~bash
git init
# add remote
git remote add --help
git remote add origin
# check remote
gh repo create cptdazlz --private
git init
# change origin name
git remote rename origin msft
git remote add origin https://github.com/cpinotossi/cptdazlz.git
git remote -v # list remotes
git status
git add .
git commit -m"protect dns via policy"
git push origin main
git add .gitignore
gh api repos/{owner}/{repo} --jq '.private'
~~~

## Powershell

~~~bash
# install powershell
sudo snap install powershell --classic
pwsh --version
~~~
