# ALZ


![High Level Deployment Flow](docs/wiki/media/high-level-deployment-flow.png)

## Deploy the modules manually

### Deploy Management Groups Module

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

### Deploy Custom Policy Definitions Module

~~~bash
# instructions
code infra-as-code/bicep/modules/policy/definitions/README.md
code infra-as-code/bicep/modules/policy/definitions/customPolicyDefinitions.bicep
code infra-as-code/bicep/modules/policy/definitions/parameters/customPolicyDefinitions.parameters.all.json
# For Azure global regions
dateYMD=$(date +%Y%m%dT%H%M%S%NZ)
NAME="alz-PolicyDefsDefaults-${dateYMD}"
LOCATION="germanywestcentral"
MGID="alz"
TEMPLATEFILE="infra-as-code/bicep/modules/policy/definitions/customPolicyDefinitions.bicep"
PARAMETERS="@infra-as-code/bicep/modules/policy/definitions/parameters/customPolicyDefinitions.parameters.all.json"

az deployment mg create --name ${NAME:0:63} --location $LOCATION --management-group-id $MGID --template-file $TEMPLATEFILE --parameters $PARAMETERS
~~~

### Deploy Custom Policy Assignments Module

After you have deployed the policy module to add all of the custom ALZ Azure Policy Definitions & Initiatives you will need to assign the modules to the relevant Management Groups as per your requirements using the [Policy Assignments module](./infra-as-code/bicep/modules/policy/assignments/README.md).

If you want to make all of the default Azure Policy Assignments that we recommend in the Azure Landing Zones conceptual architecture and reference implementation you can use the [ALZ Default Policy Assignments module](./infra-as-code/bicep/modules/policy/assignments/alzDefaults/README.md) to do this for you👍

The following example will just assign one policy:

~~~bash
code infra-as-code/bicep/modules/policy/assignments/README.md
# copy the deny example
# cp infra-as-code/bicep/modules/policy/assignments/parameters/policyAssignmentManagementGroup.deny.parameters.min.json infra-as-code/bicep/modules/policy/assignments/parameters/policyAssignmentManagementGroup.deny.pubip.parameters.min.json
code infra-as-code/bicep/modules/policy/assignments/parameters/policyAssignmentManagementGroup.deny.pubip.parameters.min.json
code infra-as-code/bicep/modules/policy/assignments/policyAssignmentManagementGroup.bicep

dateYMD=$(date +%Y%m%dT%H%M%S%NZ)
NAME="alz-alz-PolicyDenyAssignmentsDeployment-${dateYMD}"

PARAMETERS="@infra-as-code/bicep/modules/policy/assignments/parameters/policyAssignmentManagementGroup.deny.pubip.parameters.min.json"
LOCATION="germanywestcentral"
MGID="alz-landingzones"
TEMPLATEFILE="infra-as-code/bicep/modules/policy/assignments/policyAssignmentManagementGroup.bicep"

az deployment mg create --name ${NAME:0:63} --location $LOCATION --management-group-id $MGID --template-file $TEMPLATEFILE --parameters $PARAMETERS
~~~

If you wish to add your own additional custom Azure Policy Definitions please review [How Does ALZ-Bicep Implement Azure Policies?](https://github.com/Azure/ALZ-Bicep/wiki/PolicyDeepDive) and more specifically [Assigning Azure Policies](https://github.com/Azure/ALZ-Bicep/wiki/AssigningPolicies)

#### alzDefaultPolicyAssignments

In case we like to apply all Policies defined by ALZ we can use the following command:

~~~bash
code infra-as-code/bicep/modules/policy/assignments/alzDefaults/README.md
code infra-as-code/bicep/modules/policy/assignments/alzDefaults/alzDefaultPolicyAssignments.bicep
code infra-as-code/bicep/modules/policy/assignments/alzDefaults/parameters/alzDefaultPolicyAssignments.parameters.all.json

dateYMD=$(date +%Y%m%dT%H%M%S%NZ)
NAME="alz-alzPolicyAssignmentDefaults-${dateYMD}"
LOCATION="eastus"
MGID="alz"
TEMPLATEFILE="infra-as-code/bicep/modules/policy/assignments/alzDefaults/alzDefaultPolicyAssignments.bicep"
PARAMETERS="@infra-as-code/bicep/modules/policy/assignments/alzDefaults/parameters/alzDefaultPolicyAssignments.parameters.all.json"

az deployment mg create --name ${NAME:0:63} --location $LOCATION --management-group-id $MGID --template-file $TEMPLATEFILE --parameters $PARAMETERS
~~~

#### Which policies at which scope have been assigned via alzDefaultPolicyAssignments.bicep?

Verify which policies have been assigned to the management group alz.

~~~bash
alzRootScope="/providers/Microsoft.Management/managementGroups/alz"
az policy assignment list --scope $alzRootScope --query "[].{scope:scope,displayName:displayName}"
az policy assignment list --scope $alzRootScope --query "[].id" -o tsv | wc -l # 13
alzLzScope="/providers/Microsoft.Management/managementGroups/alz-landingzones"
az policy assignment list --scope $alzLzScope  --query "[].id" -o tsv | wc -l # 16
alzLzCorpScope="/providers/Microsoft.Management/managementGroups/alz-landingzones-corp"
az policy assignment list --scope $alzLzCorpScope  --query "[].id" -o tsv | wc -l # 5
alzLzOnlineScope="/providers/Microsoft.Management/managementGroups/alz-landingzones-online"
az policy assignment list --scope $alzLzOnlineScope  --query "[].id" -o tsv | wc -l # 0
~~~

Let us have a closer look at the once assigned to the alz-landingzone-corp management group.

~~~bash
az policy assignment list --scope $alzLzCorpScope  --query "[].name"
~~~

~~~json
[
  "Audit-PeDnsZones",
  "Deny-Public-Endpoints",
  "Deny-HybridNetworking",
  "Deny-Public-IP-On-NIC",
  "Deploy-Private-DNS-Zones"
]
~~~

Let us verify how the decision has been taken to create the assignment for the policy "Deploy-Private-DNS-Zones".

#### The Policy Set/Initative "Deploy-Private-DNS-Zones"

But before we do this let us have a look at the policy itself behind the assignment.

~~~bash
# List all assignments under the management group alz-landingzones-corp
az policy assignment list --scope $alzLzCorpScope  --query "[?contains(name,'Deploy-Private-DNS-Zones')].displayName" -o tsv # like shown at the azure portal.
# Get the id of the policy assignment "Deploy-Private-DNS-Zones"
policyDeployPDnsId=$(az policy assignment list --scope $alzLzCorpScope  --query "[?contains(name,'Deploy-Private-DNS-Zones')].policyDefinitionId" -o tsv) # PolicySetDefinition Deploy-Private-DNS-Zones
# Extract the Name from the Id of the policy assignment "Deploy-Private-DNS-Zones"
policyDeployPDnsName=$(basename "$policyDeployPDnsId")
# "deploy-private-dns-zones" is an policy set which consist of multiple policies
az policy set-definition show --management-group alz --name $policyDeployPDnsName --query "policyDefinitions[].{policyDefinitionId:policyDefinitionId,policyDefinitionReferenceId:policyDefinitionReferenceId}"
az policy set-definition show --management-group alz --name $policyDeployPDnsName --query "policyDefinitions[].policyDefinitionReferenceId" -o tsv # DINE-Private-DNS-Azure-*
# count the policies
az policy set-definition show --management-group alz --name $policyDeployPDnsName --query "policyDefinitions[].policyDefinitionReferenceId" -o tsv | wc -l # 49
~~~

Let us have a closer look at the DINE-Private-DNS-Azure-Storage-Blob policy.

We can find the string "DINE-Private-DNS-Azure-Storage-Blob" inside the following files 4 files insid our repo:

- [customPolicyDefinitions.bicep](./infra-as-code/bicep/modules/policy/definitions/customPolicyDefinitions.bicep).
- [policy_set_definitions/_policySetDefinitionsBicepInput.txt](./infra-as-code/bicep/modules/policy/definitions/lib/policy_set_definitions/_policySetDefinitionsBicepInput.txt).
- [policy_set_definition_es_Deploy-Private-DNS-Zones.parameters.json](./infra-as-code/bicep/modules/policy/definitions/lib/policy_set_definitions/policy_set_definition_es_Deploy-Private-DNS-Zones.parameters.json)
- [policy_set_definition_es_Deploy-Private-DNS-Zones.json](./infra-as-code/bicep/modules/policy/definitions/lib/policy_set_definitions/policy_set_definition_es_Deploy-Private-DNS-Zones.json)

##### customPolicyDefinitions.bicep

customPolicyDefinitions.bicep is deployed via zone0.yml.
But the assignment of the policy "Deploy-Private-DNS-Zones" is done via zone1.yml.

~~~bash
code ./infra-as-code/bicep/modules/policy/definitions/customPolicyDefinitions.bicep
~~~

Contains the following section:

~~~bicep
// This variable contains a number of objects that load in the custom Azure Policy Set/Initiative Defintions that are provided as part of the ESLZ/ALZ reference implementation - this is automatically created in the file 'infra-as-code\bicep\modules\policy\lib\policy_set_definitions\_policySetDefinitionsBicepInput.txt' via a GitHub action, that runs on a daily schedule, and is then manually copied into this variable.
var varCustomPolicySetDefinitionsArray = [
  ...
  {
    name: 'Deploy-Private-DNS-Zones'
    libSetDefinition: loadJsonContent('lib/policy_set_definitions/policy_set_definition_es_Deploy-Private-DNS-Zones.json')
    libSetChildDefinitions: [
    ...
    {
      definitionReferenceId: 'DINE-Private-DNS-Azure-Storage-Blob'
      definitionId: '/providers/Microsoft.Authorization/policyDefinitions/75973700-529f-4de2-b794-fb9b6781b6b0'
      definitionParameters: varPolicySetDefinitionEsDeployPrivateDNSZonesParameters['DINE-Private-DNS-Azure-Storage-Blob'].parameters
      definitionGroups: []
    }
    ...
    ]
  }
  ...
]
~~~

> NOTE: "/providers/Microsoft.Authorization/policyDefinitions/75973700-529f-4de2-b794-fb9b6781b6b0" does reference the build-in policy "Configure a private DNS Zone ID for blob groupID" (displayName).

~~~bash
policyDeployPDnsBlobName=$(basename "/providers/Microsoft.Authorization/policyDefinitions/75973700-529f-4de2-b794-fb9b6781b6b0")
az policy definition show --management-group alz --name $policyDeployPDnsBlobName
~~~

The build-in Policy "Configure a private DNS Zone ID for blob groupID"  parameters:
- privateDnsZoneId
- effect

The variable varPolicySetDefinitionEsDeployPrivateDNSZonesParameters is defind as follow:

~~~bicep
var varPolicySetDefinitionEsDeployPrivateDNSZonesParameters = loadJsonContent('lib/policy_set_definitions/policy_set_definition_es_Deploy-Private-DNS-Zones.parameters.json')
~~~

##### policy_set_definition_es_Deploy-Private-DNS-Zones.parameters.json

~~~bash
code infra-as-code/bicep/modules/policy/definitions/lib/policy_set_definitions/policy_set_definition_es_Deploy-Private-DNS-Zones.parameters.json
~~~

Contains the following section:

~~~json
  "DINE-Private-DNS-Azure-Storage-Blob": {
    "parameters": {
      "privateDnsZoneId": {
        "value": "[[parameters('azureStorageBlobPrivateDnsZoneId')]"
      },
      "effect": {
        "value": "[[parameters('effect')]"
      }
    }
  },
~~~


##### policy_set_definition_es_Deploy-Private-DNS-Zones.json

~~~bash
code infra-as-code/bicep/modules/policy/definitions/lib/policy_set_definitions/policy_set_definition_es_Deploy-Private-DNS-Zones.json
~~~

Referenced inside customPolicyDefinitions.bicep:

~~~bicep
var varCustomPolicySetDefinitionsArray = [
  ...
  {
    name: 'Deploy-Private-DNS-Zones'
    libSetDefinition: loadJsonContent('lib/policy_set_definitions/policy_set_definition_es_Deploy-Private-DNS-Zones.json')
    libSetChildDefinitions: [
~~~

Contains the following section:

~~~json
"parameters": {
  ...
  "azureStorageBlobPrivateDnsZoneId": {
    "type": "string",
    "defaultValue": "",
    "metadata": {
      "displayName": "azureStorageBlobPrivateDnsZoneId",
      "strongType": "Microsoft.Network/privateDnsZones",
      "description": "Private DNS Zone Identifier"
    }
  },
  ...
  "effect": {
    "type": "string",
    "metadata": {
      "displayName": "Effect",
      "description": "Enable or disable the execution of the policy"
    },
    "allowedValues": [
      "DeployIfNotExists",
      "Disabled"
    ],
    "defaultValue": "DeployIfNotExists"
  },
  ...
}
"policyDefinitions": [
  ...
          {
        "policyDefinitionReferenceId": "DINE-Private-DNS-Azure-Storage-Blob",
        "policyDefinitionId": "/providers/Microsoft.Authorization/policyDefinitions/75973700-529f-4de2-b794-fb9b6781b6b0",
        "parameters": {
          "privateDnsZoneId": {
            "value": "[[parameters('azureStorageBlobPrivateDnsZoneId')]"
          },
          "effect": {
            "value": "[[parameters('effect')]"
          }
        },
        "groupNames": []
      },
...
]
~~~

##### Service Principal of the policy "Deploy-Private-DNS-Zones"

The DINE policy "Configure a private DNS Zone ID for blob groupID" needs an Service Principal to be able to modify our private DNS Zone.

~~~bash
# List all Service Principals assignmnets for the management group alz scope
alzLzCorpScope="/providers/Microsoft.Management/managementGroups/alz-landingzones-corp"
policyIdentityId=$(az policy assignment list --scope $alzLzCorpScope --query "[?name=='Deploy-Private-DNS-Zones'].identity.principalId" -o tsv)
az ad sp show --id $policyIdentityId
az role assignment list --assignee $policyIdentityId # Network Contributor
~~~

> NOTE: The Service Principal scope is set to the subscription which does contain the private DNS Zone.

##### Conclusion

The policy "Deploy-Private-DNS-Zones" is
- referenced in customPolicyDefinitions.bicep, inside the variable varCustomPolicySetDefinitionsArray which does reference
 - libSetDefinition
 - varPolicySetDefinitionEsDeployPrivateDNSZonesParameters which does reference policy_set_definition_es_Deploy-Private-DNS-Zones.parameters.json
 - which does reference policy_set_definition_es_Deploy-Private-DNS-Zones.json.


#### Wher does the assignment of the policy "Deploy-Private-DNS-Zones" happen?
After we clarified how the policy set has been defined in our tenant (via customPolicyDefinitions.bicep) we like to understand how the assignment is done.

The assignment is done inside zone1.yml:

~~~yml
- name: Deploy Default Policy Assignments
  id: create_policy_assignments
  uses: azure/arm-deploy@v1
  with:
    scope: managementgroup
    managementGroupId: ${{ env.ManagementGroupPrefix }}
    region: ${{ vars.LOCATION_GWC }}
    template: infra-as-code/bicep/modules/policy/assignments/alzDefaults/alzDefaultPolicyAssignments.bicep
    parameters: infra-as-code/bicep/modules/policy/assignments/alzDefaults/parameters/alzDefaultPolicyAssignments.parameters.all.json parLogAnalyticsWorkspaceResourceId=${{ vars.LAW_ID_GWC }} parDdosProtectionPlanId="" parPrivateDnsResourceGroupId=${{ vars.PDNS_RG_ID }} parLogAnalyticsWorkSpaceAndAutomationAccountLocation=${{ vars.LOCATION_GWC }}
    deploymentName: create_policy_assignments-${{ env.runNumber }}
    failOnStdErr: false
~~~

##### alzDefaultPolicyAssignments.bicep
~~~bash
code ./infra-as-code/bicep/modules/policy/assignments/alzDefaults/alzDefaultPolicyAssignments.bicep
~~~

Let us try to find our policy "Deploy-Private-DNS-Zones" inside the bicep file.
In additon let us try to focus on Storage Blob.

~~~bicep
@sys.description('Resource ID of the Resource Group that conatin the Private DNS Zones. If left empty, the policy Deploy-Private-DNS-Zones will not be assigned to the corp Management Group.')
param parPrivateDnsResourceGroupId string = ''
...
var varPolicyAssignmentDeployPrivateDNSZones = {
  definitionId: '${varTopLevelManagementGroupResourceId}/providers/Microsoft.Authorization/policySetDefinitions/Deploy-Private-DNS-Zones'
  libDefinition: loadJsonContent('../../../policy/assignments/lib/policy_assignments/policy_assignment_es_deploy_private_dns_zones.tmpl.json')
}
...
// Deploy-Private-DNS-Zones Variables

var varPrivateDnsZonesResourceGroupSubscriptionId = !empty(parPrivateDnsResourceGroupId) ? split(parPrivateDnsResourceGroupId, '/')[2] : ''

var varPrivateDnsZonesBaseResourceId = '${parPrivateDnsResourceGroupId}/providers/Microsoft.Network/privateDnsZones/'

var varPrivateDnsZonesFinalResourceIds = {
  ...
  azureStorageBlobPrivateDnsZoneId: '${varPrivateDnsZonesBaseResourceId}privatelink.blob.core.windows.net'
  ...
}
...
// Module - Policy Assignment - Deploy-Private-DNS-Zones
module modPolicyAssignmentConnDeployPrivateDnsZones '../../../policy/assignments/policyAssignmentManagementGroup.bicep' = [for mgScope in varCorpManagementGroupIdsFiltered: if ((!empty(varPrivateDnsZonesResourceGroupSubscriptionId)) && (!contains(parExcludedPolicyAssignments, varPolicyAssignmentDeployPrivateDNSZones.libDefinition.name)) && parLandingZoneChildrenMgAlzDefaultsEnable) {
  scope: managementGroup(mgScope)
  name: contains(mgScope, 'confidential') ? varModuleDeploymentNames.modPolicyAssignmentLzsConfidentialCorpDeployPrivateDnsZones : varModuleDeploymentNames.modPolicyAssignmentLzsCorpDeployPrivateDnsZones
  params: {
    parPolicyAssignmentDefinitionId: varPolicyAssignmentDeployPrivateDNSZones.definitionId
    parPolicyAssignmentName: varPolicyAssignmentDeployPrivateDNSZones.libDefinition.name
    parPolicyAssignmentDisplayName: varPolicyAssignmentDeployPrivateDNSZones.libDefinition.properties.displayName
    parPolicyAssignmentDescription: varPolicyAssignmentDeployPrivateDNSZones.libDefinition.properties.description
    parPolicyAssignmentParameters: varPolicyAssignmentDeployPrivateDNSZones.libDefinition.properties.parameters
    parPolicyAssignmentParameterOverrides: {
      ...
      azureStorageBlobPrivateDnsZoneId: {
        value: varPrivateDnsZonesFinalResourceIds.azureStorageBlobPrivateDnsZoneId
      }
      ...
    }
    parPolicyAssignmentIdentityType: varPolicyAssignmentDeployPrivateDNSZones.libDefinition.identity.type
    parPolicyAssignmentEnforcementMode: parDisableAlzDefaultPolicies ? 'DoNotEnforce' : varPolicyAssignmentDeployPrivateDNSZones.libDefinition.properties.enforcementMode
    parPolicyAssignmentIdentityRoleDefinitionIds: [
      varRbacRoleDefinitionIds.networkContributor
    ]
    parPolicyAssignmentIdentityRoleAssignmentsSubs: [
      varPrivateDnsZonesResourceGroupSubscriptionId
    ]
    parTelemetryOptOut: parTelemetryOptOut
  }
}]
...
~~~







-----------------------------------------------------------------------------------
Lets verify which DNS Management Policy Assignments are available in the ALZ Root Management Group.

~~~bash
az policy definition list --management-group alz --query "[?contains(displayName,'DNS')].displayName" -o tsv | wc -l # 53 policies
az policy assignment list --scope $alzRootScope  --query "[?contains(displayName,'DNS')]" -o tsv | wc -l # 0, no match
~~~

Out of all the policies which are defined at the root MG scope of our Landing Zone none of them are assigned to the management group alz right now.
Let us try to figure out why they are not assigned.

To deploy all LZ relevant policies we did follow the instruction mentioned in the [ALZ Default Policy Assignments module README.md](./infra-as-code/bicep/modules/policy/assignments/alzDefaults/README.md).

It does refer the Bicep Configuration File [alzDefaultPolicyAssignments.bicep](./infra-as-code/bicep/modules/policy/assignments/alzDefaults/alzDefaultPolicyAssignments.bicep).

The alzDefaultPolicyAssignments.bicep contains a long list of policies which will be assigned.
During the deployment we used the parameter file [alzDefaultPolicyAssignments.parameters.all.json](./infra-as-code/bicep/modules/policy/assignments/alzDefaults/parameters/alzDefaultPolicyAssignments.parameters.all.json) referenced in the github workflow definition [zone-1.yml](./.github/workflows/zone-1.yml).

But we did overwrite some of the parameters:
- parLogAnalyticsWorkspaceResourceId=${{ vars.LAW_ID_GWC }}
- parDdosProtectionPlanId=""
- parPrivateDnsResourceGroupId=${{ vars.PDNS_RG_ID }}
- parLogAnalyticsWorkSpaceAndAutomationAccountLocation=${{ vars.LOCATION_GWC }}
- deploymentName: create_policy_assignments-${{ env.runNumber }

So we need to focus on one policy which is relevatn to us and which we would like to see to become part (assigned) of our governance.

A good one would be the policy "Deny the creation of private DNS" (displayName).

We expect this policy to be assigned on the Management Group which does contain all Landing Zone Subscriptions. In our case this would be one of the folllowing Management Groups:

- alz-landingzones
- alz-landingzones-corp
- alz-landingzones-online

But as we can see via the following azure cli command it is not assigned to the management groups mentioned above.

~~~bash
policyDnsDenyName=$(az policy definition list --management-group alz --query "[?contains(displayName,'Deny the creation of private DNS')].name" -o tsv)
az policy definition show --management-group alz --name $policyDnsDenyName
policyDnsDenyId=$(az policy definition show --management-group alz --name $policyDnsDenyName --query id -o tsv)
alzRootScope="/providers/Microsoft.Management/managementGroups/alz"
az policy assignment list --scope $alzRootScope  --query "[].policyDefinitionId"
az policy assignment list --scope $alzRootScope  --query "[?contains(policyDefinitionId,'$policyDnsDenyId')].policyDefinitionId" # no match
alzLzScope="/providers/Microsoft.Management/managementGroups/alz-landingzones"
az policy assignment list --scope $alzLzScope  --query "[?contains(policyDefinitionId,'$policyDnsDenyId')].policyDefinitionId" # no match
alzLzCorpScope="/providers/Microsoft.Management/managementGroups/alz-landingzones-corp"
az policy assignment list --scope $alzLzCorpScope  --query "[?contains(policyDefinitionId,'$policyDnsDenyId')].policyDefinitionId" # no match
alzLzOnlineScope="/providers/Microsoft.Management/managementGroups/alz-landingzones-online"
az policy assignment list --scope $alzLzOnlineScope  --query "[?contains(policyDefinitionId,'$policyDnsDenyId')].policyDefinitionId" # no match
~~~

Now let´s try to figure out why the policy is not assigned.

Maybe this police does get assigned directly to Landing Zone subscriptions. If this is the case, this would explain why we do not see any assignment for the policy so far on the management group level.

But before creating a Landing Zone we would like to verify if our current understanding is correct.

What are the condition to assign the policy "Deny the creation of private DNS" (displayName) in the bicep file [alzDefaultPolicyAssignments.bicep](./infra-as-code/bicep/modules/policy/assignments/alzDefaults/alzDefaultPolicyAssignments.bicep)?

The policy is defined in the file [policy_definition_es_Deny-Private-DNS-Zones.json](./infra-as-code/bicep/modules/policy/definitions/lib/policy_definitions/policy_definition_es_Deny-Private-DNS-Zones.json).

The only reference to the policy is inside the file (customPolicyDefinitions.bicep)[./infra-as-code/bicep/modules/policy/definitions/customPolicyDefinitions.bicep].



~~~bash
# Find the policy definition file inside our repo.
ls /infra-as-code/bicep/modules/policy/definitions/lib/policy_definitions/policy_definition_es_Deny-Private-DNS-Zones.json

grep "policy/definitions/lib/policy_definitions/" ./infra-as-code/bicep/modules/policy/assignments/alzDefaults/alzDefaultPolicyAssignments.bicep # no match

# Find Deny-Private-DNS-Zone inside the bicep file
grep -A 10 -B 10 "policy_definition_es_Deny-Private-DNS-Zones.json" ./infra-as-code/bicep/modules/policy/assignments/alzDefaults/alzDefaultPolicyAssignments.bicep # no match
~~~

Let us try to find the policy instruction based on the json policy definition name

~~~bash
dnsPolicyFilePath=$(ls ./infra-as-code/bicep/modules/policy/definitions/lib/policy_definitions/*DNS*)
dnsPolicyFileName=$(basename "$dnsPolicyFilePath")
echo $dnsPolicyFileName
grep -A 10 -B 10 "$dnsPolicyFileName" ./infra-as-code/bicep/modules/policy/assignments/alzDefaults/alzDefaultPolicyAssignments.bicep # no match
~~~

Based on Manual research I found the following section:

~~~bicep
// Module - Policy Assignment - Deploy-Private-DNS-Zones
module modPolicyAssignmentConnDeployPrivateDnsZones '../../../policy/assignments/policyAssignmentManagementGroup.bicep' = [for mgScope in varCorpManagementGroupIdsFiltered: if ((!empty(varPrivateDnsZonesResourceGroupSubscriptionId)) && (!contains(parExcludedPolicyAssignments, varPolicyAssignmentDeployPrivateDNSZones.libDefinition.name)) && parLandingZoneChildrenMgAlzDefaultsEnable) {
  scope: managementGroup(mgScope)
  name: contains(mgScope, 'confidential') ? varModuleDeploymentNames.modPolicyAssignmentLzsConfidentialCorpDeployPrivateDnsZones : varModuleDeploymentNames.modPolicyAssignmentLzsCorpDeployPrivateDnsZones
  params: {
    parPolicyAssignmentDefinitionId: varPolicyAssignmentDeployPrivateDNSZones.definitionId
    parPolicyAssignmentName: varPolicyAssignmentDeployPrivateDNSZones.libDefinition.name
    parPolicyAssignmentDisplayName: varPolicyAssignmentDeployPrivateDNSZones.libDefinition.properties.displayName
    parPolicyAssignmentDescription: varPolicyAssignmentDeployPrivateDNSZones.libDefinition.properties.description
    parPolicyAssignmentParameters: varPolicyAssignmentDeployPrivateDNSZones.libDefinition.properties.parameters
    parPolicyAssignmentParameterOverrides: {
      azureFilePrivateDnsZoneId: {
        value: varPrivateDnsZonesFinalResourceIds.azureFilePrivateDnsZoneId
      }
      // long list of all pDNS Zones
    }
    parPolicyAssignmentIdentityType: varPolicyAssignmentDeployPrivateDNSZones.libDefinition.identity.type
    parPolicyAssignmentEnforcementMode: parDisableAlzDefaultPolicies ? 'DoNotEnforce' : varPolicyAssignmentDeployPrivateDNSZones.libDefinition.properties.enforcementMode
    parPolicyAssignmentIdentityRoleDefinitionIds: [
      varRbacRoleDefinitionIds.networkContributor
    ]
    parPolicyAssignmentIdentityRoleAssignmentsSubs: [
      varPrivateDnsZonesResourceGroupSubscriptionId
    ]
    parTelemetryOptOut: parTelemetryOptOut
  }
}]

code ./infra-as-code/bicep/modules/policy/assignments/alzDefaults/alzDefaultPolicyAssignments.bicep #line 1187
~~~

Let us try to understand each instruction.

> ***module modPolicyAssignmentConnDeployPrivateDnsZones '../../../policy/assignments/policyAssignmentManagementGroup.bicep'***= [for mgScope in varCorpManagementGroupIdsFiltered: if ((!empty(varPrivateDnsZonesResourceGroupSubscriptionId)) && (!contains(parExcludedPolicyAssignments, varPolicyAssignmentDeployPrivateDNSZones.libDefinition.name)) && parLandingZoneChildrenMgAlzDefaultsEnable)

The creation of the assignments is definewd in a seperate bicep file.

> module modPolicyAssignmentConnDeployPrivateDnsZones '../../../policy/assignments/policyAssignmentManagementGroup.bicep'= [***for mgScope in varCorpManagementGroupIdsFiltered***: if ((!empty(varPrivateDnsZonesResourceGroupSubscriptionId)) && (!contains(parExcludedPolicyAssignments, varPolicyAssignmentDeployPrivateDNSZones.libDefinition.name)) && parLandingZoneChildrenMgAlzDefaultsEnable)

We iterate over the variable varCorpManagementGroupIdsFiltered.

In general the policy assignments will only apply to the Management Groups varCorpManagementGroupIdsFiltered. Like the name mentioned, this is an variable which is defined as follow:

~~~bash
var varCorpManagementGroupIdsFiltered = parLandingZoneMgConfidentialEnable ? varCorpManagementGroupIds : filter(varCorpManagementGroupIds, mg => !contains(toLower(mg), 'confidential'))

code ./infra-as-code/bicep/modules/policy/assignments/alzDefaults/alzDefaultPolicyAssignments.bicep #line 362
~~~

It does reference the array variable varCorpManagementGroupIds which is defined as follow:

~~~bash
var varCorpManagementGroupIds = [
  varManagementGroupIds.landingZonesCorp
  varManagementGroupIds.landingZonesConfidentialCorp
]
~~~

varManagementGroupIds.landingZonesCorp is a property of the object variable which is defined as follow:

~~~bash
// Management Groups Variables - Used For Policy Assignments
var varManagementGroupIds = {
  ...
  landingZonesCorp: '${parTopLevelManagementGroupPrefix}-landingzones-corp${parTopLevelManagementGroupSuffix}'
  ...
  landingZonesConfidentialCorp: '${parTopLevelManagementGroupPrefix}-landingzones-confidential-corp${parTopLevelManagementGroupSuffix}'
  ...
}
~~~

Variable varCorpManagementGroupIds
Based on my current understanding we are going to assign the policies to alz-landingzones-corp and alz-landingzones-confidential-corp.

If we lookup the assignments we will figure out that our DNS Policy is not assigned

~~~bash
# List all Service Principals assignmnets for the management group alz scope
alzCorpScope="/providers/Microsoft.Management/managementGroups/alz-landingzones-corp" #just use the management group id not the real path inside the management group tree structure.
az policy assignment list --scope $alzCorpScope --query "[?contains(displayName,'DNS')].displayName" -o tsv | wc -l # 2 policies/initiatives
az policy assignment list --scope $alzCorpScope --query "[?contains(displayName,'DNS')].{scope:scope,displayName:displayName}"
~~~

Why is the policy




module modPolicyAssignmentConnDeployPrivateDnsZones '../../../policy/assignments/policyAssignmentManagementGroup.bicep'= [for mgScope in varCorpManagementGroupIdsFiltered: if ((!empty(varPrivateDnsZonesResourceGroupSubscriptionId)) && (!contains(parExcludedPolicyAssignments, varPolicyAssignmentDeployPrivateDNSZones.libDefinition.name)) && parLandingZoneChildrenMgAlzDefaultsEnable)






~~~bash
code infra-as-code/bicep/modules/policy/assignments/alzDefaults/README.md
code infra-as-code/bicep/modules/policy/assignments/alzDefaults/alzDefaultPolicyAssignments.bicep
code infra-as-code/bicep/modules/policy/assignments/alzDefaults/parameters/alzDefaultPolicyAssignments.parameters.all.json
~~~




### Deploy custom RBAC Role definition

~~~bash
# instructions
code infra-as-code/bicep/modules/customRoleDefinitions/README.md
code infra-as-code/bicep/modules/customRoleDefinitions/customRoleDefinitions.bicep
code infra-as-code/bicep/modules/customRoleDefinitions/parameters/customRoleDefinitions.parameters.all.json
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

#### RoleAssignments

> NOT DONE !!

~~~bash
code infra-as-code/bicep/modules/roleAssignments/README.md
code infra-as-code/bicep/modules/roleAssignments/roleAssignmentManagementGroup.bicep
code infra-as-code/bicep/modules/roleAssignments/parameters/roleAssignmentManagementGroup.servicePrincipal.parameters.all.json
code infra-as-code/bicep/modules/roleAssignments/generateddocs/roleAssignmentManagementGroup.bicep.md
~~~

### Deploy Logging & Sentinel Module

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

### Deploy Management Groups Diagnostic Settings Module

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


### Deploy vWAN Network Module

The module seesm very basic. The vWAN private DNS limitation is not covered. So you will need to provide an virtual Network to which the private DNS will be linked.
That network will be shared via the

~~~bash
# instructions
code ./infra-as-code/bicep/modules/vwanConnectivity/README.md
code ./infra-as-code/bicep/modules/vwanConnectivity/vwanConnectivity.bicep
code ./infra-as-code/bicep/modules/vwanConnectivity/parameters/vwanConnectivity.parameters.my.json

# For Azure global regions
# Set Platform connectivity subscription ID as the the current subscription
ConnectivitySubscriptionId=$(az account list --query "[?name=='sub-myedge-01'].id" -o tsv)
az account set --subscription $ConnectivitySubscriptionId
# we are going to use our own parameters file
cp infra-as-code/bicep/modules/vwanConnectivity/parameters/vwanConnectivity.parameters.all.json infra-as-code/bicep/modules/vwanConnectivity/parameters/vwanConnectivity.parameters.my.json
code ./infra-as-code/bicep/modules/vwanConnectivity/parameters/vwanConnectivity.parameters.my.json

# Set the top level MG Prefix in accordance to your environment. This example assumes default 'alz'.
TopLevelMGPrefix="alz"
dateYMD=$(date +%Y%m%dT%H%M%S%NZ)
NAME="alz-vwanConnectivityDeploy-${dateYMD}"
GROUP="rg-$TopLevelMGPrefix-vwan-001"
TEMPLATEFILE="infra-as-code/bicep/modules/vwanConnectivity/vwanConnectivity.bicep"
PARAMETERS="@infra-as-code/bicep/modules/vwanConnectivity/parameters/vwanConnectivity.parameters.my.json"
LOCATION="germanywestcentral"
MGID="alz"

az group create --location $LOCATION --name $GROUP
az deployment group create --name ${NAME:0:63} --resource-group $GROUP --template-file $TEMPLATEFILE --parameters $PARAMETERS
~~~

## Github Actions

Setup azure credentials for github actions based on [Use GitHub Actions to connect to Azure](https://learn.microsoft.com/en-us/azure/developer/github/connect-from-azure) without secrets.

~~~bash
# define the prefix
prefix=cptdazlz
# create service principal
# Important you cannot use the following scope:
az ad sp create-for-rbac -n $prefix --role owner --query password -o tsv --scope /providers/Microsoft.Management/managementGroups/myedge
# Instead you will need to provide the root scope see also https://github.com/Azure/ALZ-Bicep/wiki/DeploymentFlow#service-principal-account
az ad sp create-for-rbac -n $prefix --role owner --query password -o tsv --scope /
# get service principal appid
appid=$(az ad sp list --display-name $prefix --query [0].appId -o tsv)
objectid=$(az ad sp list --display-name $prefix --query [0].id -o tsv)
# all github to create tokens via inpersonation of the service principal
az ad app federated-credential create --id $appid --parameters ./github.action/credential.json
# verify federation setup
az ad app federated-credential list --id $appid
~~~

We are going to provide some secretes and variables via the github build in secrets and variables feature.
> NOTE: If it comes to secret, this is a best practices, but I am unsure if variable should go into github. It would be better to keep them all at one place. Maybe the bicep parameter files are a better place. But in this case we would need to store resource ids in clear text.
~~~bash
# create github secret via gh cli for repo cptdazlz
gh secret set AZURE_CLIENT_ID -b $appid -e production
tid=$(az account show --query tenantId -o tsv)
gh secret set AZURE_TENANT_ID -b $tid -e production
subid=$(az account show --query id -o tsv)
gh secret set AZURE_SUBSCRIPTION_ID -b $subid -e production
gh secret set AZURE_OBJECT_ID -b $objectid -e production
gh secret list --env production

gh variable set MG_PREFIX -b alz --env production
gh variable set MG_TOPLEVEL_DISPLAYNAME -b "MyEdge Landing Zones" --env production
gh variable set LOCATION_GWC -b "germanywestcentral" --env production
# get log analytics workspace id
lawid=$(az monitor log-analytics workspace show -n alz-log-analytics -g rg-alz-logging-001 --query id -o tsv)
gh variable set LAW_ID_GWC -b $lawid --env production
pDnsRg=$(az group show -n rg-alz-vwan-001 --query id -o tsv)
gh variable set PDNS_RG_ID -b $pDnsRg --env production
gh variable list --env production
~~~

You will need to create an github action yaml like I did here: .github/workflows/login-test.yml

~~~bash
# trigger the login test action
gh api /repos/cpinotossi/$prefix/actions/workflows/login-test.yml/dispatches -f ref=main
# List all runs for a repository
gh run list --repo cpinotossi/$prefix
# View the details of the last run
gh run view $(gh run list --repo cpinotossi/$prefix --json databaseId --jq '.[0].databaseId') --repo cpinotossi/$prefix --log

# trigger the lz zone-1.yml action
gh api /repos/cpinotossi/$prefix/actions/workflows/zone-1.yml/dispatches -f ref=main
# List all runs for a repository
gh run list --repo cpinotossi/$prefix
# View the details of the last run
gh run view $(gh run list --repo cpinotossi/$prefix --json databaseId --jq '.[0].databaseId') --repo cpinotossi/$prefix --log-failed
gh run view -h
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



## github client

~~~bash
gh version # 2.6.0-15-g1a10fd5a (2022-03-16)
type -p curl >/dev/null || (sudo apt update && sudo apt install curl -y)
curl -fsSL https://cli.github.com/packages/githubcli-archive-keyring.gpg | sudo dd of=/usr/share/keyrings/githubcli-archive-keyring.gpg && sudo chmod go+r /usr/share/keyrings/githubcli-archive-keyring.gpg && echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" | sudo tee /etc/apt/sources.list.d/github-cli.list > /dev/null && sudo apt update && sudo apt install gh -y
sudo apt update
sudo apt install gh
which gh
whereis gh
gh version # 2.6.0-15-g1a10fd5a (2022-03-16)
sudo snap remove gh
echo $PATH
code ~/.bashrc
# add the following line export PATH=$PATH:/usr/bin/gh
source ~/.bashrc
echo $PATH
gh version # 2.40.1 (2023-12-13)
~~~

