---
title: Azure Monitor Logs Dedicated Clusters
description: Customers who ingest more than 1 TB a day of monitoring data may use dedicated rather than shared clusters
ms.subservice: logs
ms.topic: conceptual
author: rboucher
ms.author: robb
ms.date: 09/16/2020
---

# Azure Monitor Logs Dedicated Clusters

Azure Monitor Logs Dedicated Clusters are a deployment option that enables advanced capabilities for Azure Monitor Logs customers. Customers with dedicated clusters can choose the workspaces to be hosted on these clusters.

The capabilities that require dedicated clusters are:

- **[Customer-managed Keys](../platform/customer-managed-keys.md)** - Encrypt the cluster data using keys that are provided and controlled by the customer.
- **[Lockbox](../platform/customer-managed-keys.md#customer-lockbox-preview)** - Customers can control Microsoft support engineers access requests for data.
- **[Double encryption](../../storage/common/storage-service-encryption.md#doubly-encrypt-data-with-infrastructure-encryption)** protects against a scenario where one of the encryption algorithms or keys may be compromised. In this case, the additional layer of encryption continues to protect your data.
- **[Multi-workspace](../log-query/cross-workspace-query.md)** - If a customer is using more than one workspace for production it might make sense to use dedicated cluster. Cross-workspace queries will run faster if all workspaces are on the same cluster. It might also be more cost effective to use dedicated cluster as the assigned capacity reservation tiers take into account all cluster ingestion and applies to all its workspaces, even if some of them are small and not eligible for capacity reservation discount.

Dedicated clusters require customers to commit using a capacity of at least 1 TB of data ingestion per day. Migration to a dedicated cluster is simple. There is no data loss or service interruption. 

> [!IMPORTANT]
> Dedicated clusters are approved and fully supported for production deployments. However, due to temporary capacity constraints, we require your pre-register to use the feature. Use your contacts into Microsoft to provide your Subscriptions IDs.

## Management 

Dedicated clusters are managed via an Azure resource that represents Azure Monitor Log clusters. All operations are done on this resource using PowerShell or the REST API.

Once the cluster is created, it can be configured and workspaces linked to it. When a workspace is linked to a cluster, new data sent to the workspace resides on the cluster. Only workspaces that are in the same region as the cluster can be linked to the cluster. Workspaces can be unliked from a cluster with some limitations. More detail on these limitations is included in this article. 

Data ingested to dedicated clusters is being encrypted twice — once at the service level using Microsoft-managed keys or [customer-managed key](../platform/customer-managed-keys.md), and once at the infrastructure level using two different encryption algorithms and two different keys. [Double encryption](../../storage/common/storage-service-encryption.md#doubly-encrypt-data-with-infrastructure-encryption) protects against a scenario where one of the encryption algorithms or keys may be compromised. In this case, the additional layer of encryption continues to protect your data. Dedicated cluster also allows you to protect your data with [Lockbox](../platform/customer-managed-keys.md#customer-lockbox-preview) control.

All operations on the cluster level require the `Microsoft.OperationalInsights/clusters/write` action permission on the cluster. This permission could be granted via the Owner or Contributor that contains the `*/write` action or via the Log Analytics Contributor role that contains the `Microsoft.OperationalInsights/*` action. For more information on Log Analytics permissions, see [Manage access to log data and workspaces in Azure Monitor](../platform/manage-access.md). 


## Cluster pricing model

Log Analytics Dedicated Clusters use a Capacity Reservation pricing model which of at least 1000 GB/day. Any usage above the reservation level will be billed at the Pay-As-You-Go rate.  Capacity Reservation pricing information is available at the [Azure Monitor pricing page]( https://azure.microsoft.com/pricing/details/monitor/).  

The cluster capacity reservation level is configured via programmatically with Azure Resource Manager using the `Capacity` parameter under `Sku`. The `Capacity` is specified in units of GB and can have values of 1000 GB/day or more in increments of 100 GB/day.

There are two modes of billing for usage on a cluster. These can be specified by the `billingType` parameter when configuring your cluster. 

1. **Cluster**: in this case (which is the default), billing for ingested data is done at the cluster level. The ingested data quantities from each workspace associated to a cluster are aggregated to calculate the daily bill for the cluster. 

2. **Workspaces**: the Capacity Reservation costs for your Cluster are attributed proportionately to the workspaces in the Cluster (after accounting for per-node allocations from [Azure Security Center](../../security-center/index.yml) for each workspace.)

If your workspace is using legacy Per Node pricing tier, when it is linked to a cluster it will be billed based on data ingested against the cluster’s Capacity Reservation, and no longer per node. Per node data allocations from Azure Security Center will continue to be applied.

More details are billing for Log Analytics dedicated clusters are available [here]( https://docs.microsoft.com/azure/azure-monitor/platform/manage-cost-storage#log-analytics-dedicated-clusters).

## Asynchronous operations and status check

Some of the configuration steps run asynchronously because they can't be completed quickly. The status in response contains can be one of the followings: 'InProgress', 'Updating', 'Deleting', 'Succeeded or 'Failed' including the error code. When using REST, the response initially returns an HTTP status code 200 (OK) and header with Azure-AsyncOperation property when accepted:

```JSON
"Azure-AsyncOperation": "https://management.azure.com/subscriptions/subscription-id/providers/Microsoft.OperationalInsights/locations/region-name/operationStatuses/operation-id?api-version=2020-08-01"
```

You can check the status of the asynchronous operation by sending a GET request to the Azure-AsyncOperation header value:

```rst
GET https://management.azure.com/subscriptions/subscription-id/providers/microsoft.operationalInsights/locations/region-name/operationstatuses/operation-id?api-version=2020-08-01
Authorization: Bearer <token>
```

## Creating a cluster

You first create cluster resources to begin creating a dedicated cluster.

The following properties must be specified:

- **ClusterName**: Used for administrative purposes. Users are not exposed to this name.
- **ResourceGroupName**: As for any Azure resource, clusters belong to a resource group. We  recommended you use a central IT resource group because clusters are usually shared by many teams in the organization. For more design considerations, review [Designing your Azure Monitor Logs deployment](../platform/design-logs-deployment.md)
- **Location**: A cluster is located in a specific Azure region. Only workspaces located in this region can be linked to this cluster.
- **SkuCapacity**: You must specify the *capacity reservation* level (sku) when creating a *cluster* resource. The *capacity reservation* level can be in the range of 1,000 GB to 3,000 GB per day. You can update it in steps of 100 later if needed. If you need capacity reservation level higher than 3,000 GB per day, contact us at LAIngestionRate@microsoft.com. For more information on cluster costs, see [Manage Costs for Log Analytics clusters](../platform/manage-cost-storage.md#log-analytics-dedicated-clusters)

After you create your *Cluster* resource, you can edit additional properties such as *sku*, *keyVaultProperties, or *billingType*. See more details below.

> [!WARNING]
> Cluster creation triggers resource allocation and provisioning. This operation can take up to an hour to complete. It is recommended to run it asynchronously.

The user account that creates the clusters must have the standard Azure resource creation permission: `Microsoft.Resources/deployments/*` and cluster write permission `(Microsoft.OperationalInsights/clusters/write)`.

### Create 

**PowerShell**

```powershell
New-AzOperationalInsightsCluster -ResourceGroupName {resource-group-name} -ClusterName {cluster-name} -Location {region-name} -SkuCapacity {daily-ingestion-gigabyte} -AsJob

# Check when the job is done
Get-Job -Command "New-AzOperationalInsightsCluster*" | Format-List -Property *
```

**REST**

*Call* 
```rst
PUT https://management.azure.com/subscriptions/<subscription-id>/resourceGroups/<resource-group-name>/providers/Microsoft.OperationalInsights/clusters/<cluster-name>?api-version=2020-08-01
Authorization: Bearer <token>
Content-type: application/json

{
  "identity": {
    "type": "systemAssigned"
    },
  "sku": {
    "name": "capacityReservation",
    "Capacity": 1000
    },
  "properties": {
    "billingType": "cluster",
    },
  "location": "<region-name>",
}
```

*Response*

Should be 200 OK and a header.

## Check cluster provisioning status

The provisioning of the Log Analytics cluster takes a while to complete. You can check the provisioning state in several ways:

- Run Get-AzOperationalInsightsCluster PowerShell command with the resource group name and check the ProvisioningState property. The value is *ProvisioningAccount* while provisioning and *Succeeded* when completed.
  ```powershell
  New-AzOperationalInsightsCluster -ResourceGroupName {resource-group-name} 
  ```

- Copy the Azure-AsyncOperation URL value from the response and follow the asynchronous operations status check.

- Send a GET request on the *Cluster* resource and look at the *provisioningState* value. The value is *ProvisioningAccount* while provisioning and *Succeeded* when completed.

   ```rst
   GET https://management.azure.com/subscriptions/<subscription-id>/resourceGroups/<resource-group-name>/providers/Microsoft.OperationalInsights/clusters/<cluster-name>?api-version=2020-08-01
   Authorization: Bearer <token>
   ```

   **Response**

   ```json
   {
     "identity": {
       "type": "SystemAssigned",
       "tenantId": "tenant-id",
       "principalId": "principal-id"
       },
     "sku": {
       "name": "capacityReservation",
       "capacity": 1000,
       "lastSkuUpdate": "Sun, 22 Mar 2020 15:39:29 GMT"
       },
     "properties": {
       "provisioningState": "ProvisioningAccount",
       "billingType": "cluster",
       "clusterId": "cluster-id"
       },
     "id": "/subscriptions/subscription-id/resourceGroups/resource-group-name/providers/Microsoft.OperationalInsights/clusters/cluster-name",
     "name": "cluster-name",
     "type": "Microsoft.OperationalInsights/clusters",
     "location": "region-name"
   }
   ```

The *principalId* GUID is generated by the managed identity service for the *Cluster* resource.

## Check workspace link status
  
Perform Get operation on the workspace and observe if *clusterResourceId* property is present in the response under *features*. A linked workspace will have the *clusterResourceId* property.

**CLI**

```azurecli
az monitor log-analytics cluster show --resource-group "resource-group-name" --name "cluster-name"
```

**PowerShell**

```powershell
Get-AzOperationalInsightsWorkspace -ResourceGroupName "resource-group-name" -Name "workspace-name"
```

**REST**

```rest
GET https://management.azure.com/subscriptions/<subscription-id>/resourcegroups/<resource-group-name>/providers/microsoft.operationalinsights/workspaces/<workspace-name>?api-version=2020-08-01
Authorization: Bearer <token>
```

## Change cluster properties

After you create your *Cluster* resource and it is fully provisioned, you can edit additional properties at the cluster level using PowerShell or REST API. Other than the properties that are available during cluster creation, additional properties can only be set after the cluster has been provisioned:

- **keyVaultProperties**: Used to configure the Azure Key Vault used to provision an [Azure Monitor customer-managed key](../platform/customer-managed-keys.md#customer-managed-key-provisioning). It contains the following parameters:  *KeyVaultUri*, *KeyName*, *KeyVersion*. 
- **billingType** - The *billingType* property determines the billing attribution for the *cluster* resource and its data:
  - **Cluster** (default) - The Capacity Reservation costs for your Cluster are attributed to the *Cluster* resource.
  - **Workspaces** - The Capacity Reservation costs for your Cluster are attributed proportionately to the workspaces in the Cluster, with the *Cluster* resource being billed some of the usage if the total ingested data for the day is under the Capacity Reservation. See [Log Analytics Dedicated Clusters](../platform/manage-cost-storage.md#log-analytics-dedicated-clusters) to learn more about the Cluster pricing model. 

> [!NOTE]
> The *billingType* property is not supported in PowerShell.
> Cluster property updates might be executed asynchronous and can take a while to complete.

**PowerShell**

```powershell
Update-AzOperationalInsightsCluster -ResourceGroupName {resource-group-name} -ClusterName {cluster-name} -KeyVaultUri {key-uri} -KeyName {key-name} -KeyVersion {key-version}
```

**REST**

> [!NOTE]
> You can update the *Cluster* resource *sku*, *keyVaultProperties* or *billingType* using PATCH.

For example: 

*Call*

```rst
PATCH https://management.azure.com/subscriptions/<subscription-id>/resourceGroups/<resource-group-name>/providers/Microsoft.OperationalInsights/clusters/<cluster-name>?api-version=2020-08-01
Authorization: Bearer <token>
Content-type: application/json

{
   "sku": {
     "name": "capacityReservation",
     "capacity": <capacity-reservation-amount-in-GB>
     },
   "properties": {
    "billingType": "cluster",
     "KeyVaultProperties": {
       "KeyVaultUri": "https://<key-vault-name>.vault.azure.net",
       "KeyName": "<key-name>",
       "KeyVersion": "<current-version>"
       }
   },
   "location":"<region-name>"
}
```

"KeyVaultProperties" contains the Key Vault key identifier details.

*Response*

200 OK and header

### Check cluster update status

The propagation of the Key identifier takes a while to complete. You can check the update state in two ways:

- Copy the Azure-AsyncOperation URL value from the response and follow the asynchronous operations status check. 

   OR

- Send a GET request on the *Cluster* resource and look at the *KeyVaultProperties* properties. Your recently updated Key identifier details should return in the response.

   A response to GET request on the *Cluster* resource should look like this when Key identifier update is complete:

   ```json
   {
     "identity": {
       "type": "SystemAssigned",
       "tenantId": "tenant-id",
       "principalId": "principle-id"
       },
     "sku": {
       "name": "capacityReservation",
       "capacity": 1000,
       "lastSkuUpdate": "Sun, 22 Mar 2020 15:39:29 GMT"
       },
     "properties": {
       "keyVaultProperties": {
         "keyVaultUri": "https://key-vault-name.vault.azure.net",
         "kyName": "key-name",
         "keyVersion": "current-version"
         },
        "provisioningState": "Succeeded",
       "billingType": "cluster",
       "clusterId": "cluster-id"
     },
     "id": "/subscriptions/subscription-id/resourceGroups/resource-group-name/providers/Microsoft.OperationalInsights/clusters/cluster-name",
     "name": "cluster-name",
     "type": "Microsoft.OperationalInsights/clusters",
     "location": "region-name"
  }
  ```

## Link a workspace to the cluster

When a workspace is linked to a dedicated cluster, new data that is ingested into the workspace is routed to the new cluster while existing data remains on the existing cluster. If the dedicated cluster is encrypted using customer-managed keys (CMK), only new data is encrypted with the key. The system is abstracting this difference from the users and the users just query the workspace as usual while the system performs cross-cluster queries on the backend.

A cluster can be linked to up to 100 workspaces. Linked workspaces are located in the same region as the cluster. To protect the system backend and avoid fragmentation of data, a workspace can’t be linked to a cluster more than twice a month.

To perform the link operation, you need to have 'write' permissions to both the workspace and the *cluster* resource:

- In the workspace: *Microsoft.OperationalInsights/workspaces/write*
- In the *cluster* resource: *Microsoft.OperationalInsights/clusters/write*

Other than the billing aspects, the linked workspace keeps its own settings such as the length of data retention.
The workspace and the cluster can be in different subscriptions. It's possible for the workspace and cluster to be in different tenants if Azure Lighthouse is used to map both of them to a single tenant.

As any cluster operation, linking a workspace can be performed only after the completion of the Log Analytics cluster provisioning.

> [!WARNING]
> Linking a workspace to a cluster requires syncing multiple backend components and assuring cache hydration. This operation may take up to two hours to complete. We recommended you run it asynchronously.


**PowerShell**

Use the following PowerShell command to link to a cluster:

```powershell
# Find cluster resource ID
$clusterResourceId = (Get-AzOperationalInsightsCluster -ResourceGroupName {resource-group-name} -ClusterName {cluster-name}).id

# Link the workspace to the cluster
Set-AzOperationalInsightsLinkedService -ResourceGroupName {resource-group-name} -WorkspaceName {workspace-name} -LinkedServiceName cluster -WriteAccessResourceId $clusterResourceId -AsJob

# Check when the job is done
Get-Job -Command "Set-AzOperationalInsightsLinkedService" | Format-List -Property *
```


**REST**

Use the following REST call to link to a cluster:

*Send*

```rst
PUT https://management.azure.com/subscriptions/<subscription-id>/resourcegroups/<resource-group-name>/providers/microsoft.operationalinsights/workspaces/<workspace-name>/linkedservices/cluster?api-version=2020-08-01 
Authorization: Bearer <token>
Content-type: application/json

{
  "properties": {
    "WriteAccessResourceId": "/subscriptions/<subscription-id>/resourcegroups/<resource-group-name>/providers/microsoft.operationalinsights/clusters/<cluster-name>"
    }
}
```

*Response*

200 OK and header.

### Using customer-managed keys with linking

If you use customer-managed keys, ingested data is stored encrypted with your managed key after the association operation, which can take up to 90 minutes to complete. 

You can check the workspace association state in two ways:

- Copy the Azure-AsyncOperation URL value from the response and follow the asynchronous operations status check.

- Send a [Workspaces – Get](/rest/api/loganalytics/workspaces/get) request and observe the response. The associated workspace has a clusterResourceId under "features".

A send request looks like the following:

*Send*

```rest
GET https://management.azure.com/subscriptions/<subscription-id>/resourcegroups/<resource-group-name>/providers/microsoft.operationalInsights/workspaces/<workspace-name>?api-version=2020-08-01
Authorization: Bearer <token>
```

*Response*

```json
{
  "properties": {
    "source": "Azure",
    "customerId": "workspace-name",
    "provisioningState": "Succeeded",
    "sku": {
      "name": "pricing-tier-name",
      "lastSkuUpdate": "Tue, 28 Jan 2020 12:26:30 GMT"
    },
    "retentionInDays": 31,
    "features": {
      "legacy": 0,
      "searchVersion": 1,
      "enableLogAccessUsingOnlyResourcePermissions": true,
      "clusterResourceId": "/subscriptions/subscription-id/resourceGroups/resource-group-name/providers/Microsoft.OperationalInsights/clusters/cluster-name"
    },
    "workspaceCapping": {
      "dailyQuotaGb": -1.0,
      "quotaNextResetTime": "Tue, 28 Jan 2020 14:00:00 GMT",
      "dataIngestionStatus": "RespectQuota"
    }
  },
  "id": "/subscriptions/subscription-id/resourcegroups/resource-group-name/providers/microsoft.operationalinsights/workspaces/workspace-name",
  "name": "workspace-name",
  "type": "Microsoft.OperationalInsights/workspaces",
  "location": "region-name"
}
```

## Unlink a workspace from a dedicated cluster

You can unlink a workspace from a cluster. After unlinking a workspace from the cluster, new data associated with this workspace is not sent to the dedicated cluster. Also, the workspace billing is no longer done via the cluster. 
Old data of the unlinked workspace might be left on the cluster. If this data is encrypted using customer-managed keys (CMK), the Key Vault secrets are kept. The system is abstracts this change from Log Analytics users. Users can just query the workspace as usual. The system performs cross-cluster queries on the backend as needed with no indication to users.  

> [!WARNING] 
> There is a limit of two linking operations for a specific workspace within a month. Take time to consider and plan unlinking actions accordingly.

## Delete a dedicated cluster

A dedicated cluster resource can be deleted. You must unlink all workspaces from the cluster before deleting it. You need 'write' permissions on the *Cluster* resource to perform this operation. 

Once the cluster resource is deleted, the physical cluster enters a purge and deletion process. Deletion of a cluster deletes all the data that was stored on the cluster. The data could be from workspaces that were linked to the cluster in the past.

A *Cluster* resource that was deleted in the last 14 days is in soft-delete state and can be recovered with its data. Since all workspaces got disassociated from the *Cluster* resource with *Cluster* resource deletion, you need to re-associate your workspaces after the recovery. The recovery operation cannot be performed by the user contact your Microsoft channel or support for recovery requests.

Within the 14 days after deletion, the cluster resource name is reserved and cannot be used by other resources.

> [!WARNING] 
> There is a limit of three clusters per subscription. Both active and soft-deleted clusters are counted as part of this. Customers should not create recurrent procedures that create and delete clusters. It has a significant impact on Log Analytics backend systems.

**PowerShell**

Use the following PowerShell command to delete a cluster:

  ```powershell
  Remove-AzOperationalInsightsCluster -ResourceGroupName "resource-group-name" -ClusterName "cluster-name"
  ```

**REST**

Use the following REST call to delete a cluster:

  ```rst
  DELETE https://management.azure.com/subscriptions/<subscription-id>/resourceGroups/<resource-group-name>/providers/Microsoft.OperationalInsights/clusters/<cluster-name>?api-version=2020-08-01
  Authorization: Bearer <token>
  ```

  **Response**

  200 OK

## Get all clusters in a resource group
  
**CLI**

```azurecli
az monitor log-analytics cluster list --resource-group "resource-group-name"
```

**PowerShell**

```powershell
Get-AzOperationalInsightsCluster -ResourceGroupName "resource-group-name"
```

**REST**

*Call*

  ```rst
  GET https://management.azure.com/subscriptions/<subscription-id>/resourcegroups/<resource-group-name>/providers/Microsoft.OperationalInsights/clusters?api-version=2020-08-01
  Authorization: Bearer <token>
  ```

*Response*
  
  ```json
  {
    "value": [
      {
        "identity": {
          "type": "SystemAssigned",
          "tenantId": "tenant-id",
          "principalId": "principal-Id"
        },
        "sku": {
          "name": "capacityReservation",
          "capacity": 1000,
          "lastSkuUpdate": "Sun, 22 Mar 2020 15:39:29 GMT"
          },
        "properties": {
           "keyVaultProperties": {
              "keyVaultUri": "https://key-vault-name.vault.azure.net",
              "keyName": "key-name",
              "keyVersion": "current-version"
              },
          "provisioningState": "Succeeded",
          "billingType": "cluster",
          "clusterId": "cluster-id"
        },
        "id": "/subscriptions/subscription-id/resourcegroups/resource-group-name/providers/microsoft.operationalinsights/workspaces/workspace-name",
        "name": "cluster-name",
        "type": "Microsoft.OperationalInsights/clusters",
        "location": "region-name"
      }
    ]
  }
  ```

## Get all clusters in a subscription

**CLI**

```azurecli
az monitor log-analytics cluster list
```

**PowerShell**

```powershell
Get-AzOperationalInsightsCluster
```

**REST**

*Call*

```rst
GET https://management.azure.com/subscriptions/<subscription-id>/providers/Microsoft.OperationalInsights/clusters?api-version=2020-08-01
Authorization: Bearer <token>
```
    
*Response*
    
The same as for 'clusters in a resource group', but in subscription scope.

## Update capacity reservation in cluster

When the data volume to your linked workspaces change over time and you want to update the capacity reservation level appropriately. The Capacity is specified in units of GB and can have values of 1000 GB/day or more in increments of 100 GB/day. Note that you don’t have to provide the full REST request body but should include the sku.

**CLI**

```azurecli
az monitor log-analytics cluster update --name "cluster-name" --resource-group "resource-group-name" --sku-capacity daily-ingestion-gigabyte
```

**PowerShell**

```powershell
Update-AzOperationalInsightsCluster -ResourceGroupName "resource-group-name" -ClusterName "cluster-name" -SkuCapacity daily-ingestion-gigabyte
```

**REST**

*Call*

  ```rst
  PATCH https://management.azure.com/subscriptions/<subscription-id>/resourceGroups/<resource-group-name>/providers/Microsoft.OperationalInsights/clusters/<cluster-name>?api-version=2020-08-01
  Authorization: Bearer <token>
  Content-type: application/json

  {
    "sku": {
      "name": "capacityReservation",
      "Capacity": 2000
    }
  }
  ```

## Update billingType in cluster

The *billingType* property determines the billing attribution for the cluster and its data:
- *cluster* (default) -- The billing is attributed to the subscription hosting your Cluster resource
- *workspaces* -- The billing is attributed to the subscriptions hosting your workspaces proportionally

**REST**

*Call*

  ```rst
  PATCH https://management.azure.com/subscriptions/<subscription-id>/resourceGroups/<resource-group-name>/providers/Microsoft.OperationalInsights/clusters/<cluster-name>?api-version=2020-08-01
  Authorization: Bearer <token>
  Content-type: application/json

  {
    "properties": {
      "billingType": "cluster",
      }  
  }
  ```

## Limits and constraints

- The max number of cluster per region and subscription is 2

- The maximum of linked workspaces to cluster is 1000

- You can link a workspace to your cluster and then unlink it. The number of workspace link operations on particular workspace is limited to 2 in a period of 30 days.

- Workspace link to cluster should be carried ONLY after you have verified that the Log Analytics cluster provisioning was completed. Data sent to your workspace prior to the completion will be dropped and won't be recoverable.

- Cluster move to another resource group or subscription isn't supported currently.

- Workspace link to cluster will fail if it is linked to another cluster.

- Lockbox isn't available in China currently. 

- [Double encryption](../../storage/common/storage-service-encryption.md#doubly-encrypt-data-with-infrastructure-encryption) is configured automatically for clusters created from October 2020 in supported regions. You can verify if your cluster is configured for Double encryption by a GET request on the cluster and observing the `"isDoubleEncryptionEnabled"` property value - it's `true` for clusters with Double encryption enabled. 
  - If you create a cluster and get an error "<region-name> doesn’t support Double Encryption for clusters.", you can still create the cluster without Double Encryption. Add `"properties": {"isDoubleEncryptionEnabled": false}` property in the REST request body.
  - Double encryption setting can not be changed after the cluster has been created.

## Troubleshooting

- If you get conflict error when creating a cluster – it may be that you have deleted your cluster in the last 14 days and it’s in a soft-delete state. The cluster name remains reserved during the soft-delete period and you can't create a new cluster with that name. The name is released after the soft-delete period when the cluster is permanently deleted.

- If you update your cluster while an operation is in progress, the operation will fail.

- Some operations are long and can take a while to complete -- these are cluster create, cluster key update and cluster delete. You can check the operation status in two ways:
  - When using REST, copy the Azure-AsyncOperation URL value from the response and follow the [asynchronous operations status check](#asynchronous-operations-and-status-check).
  - Send GET request to cluster or workspace and observe the response. For example, unlinked workspace won't have the *clusterResourceId* under *features*.

- Error messages
  
  Cluster Create:
  -  400 -- Cluster name is not valid. Cluster name can contain characters a-z, A-Z, 0-9 and length of 3-63.
  -  400 -- The body of the request is null or in bad format.
  -  400 -- SKU name is invalid. Set SKU name to capacityReservation.
  -  400 -- Capacity was provided but SKU is not capacityReservation. Set SKU name to capacityReservation.
  -  400 -- Missing Capacity in SKU. Set Capacity value to 1000 or higher in steps of 100 (GB).
  -  400 -- Capacity in SKU is not in range. Should be minimum 1000 and up to the max allowed capacity which is available under ‘Usage and estimated cost’ in your workspace.
  -  400 -- Capacity is locked for 30 days. Decreasing capacity is permitted 30 days after update.
  -  400 -- No SKU was set. Set the SKU name to capacityReservation and Capacity value to 1000 or higher in steps of 100 (GB).
  -  400 -- Identity is null or empty. Set Identity with systemAssigned type.
  -  400 -- KeyVaultProperties are set on creation. Update KeyVaultProperties after cluster creation.
  -  400 -- Operation cannot be executed now. Async operation is in a state other than succeeded. Cluster must complete its operation before any update operation is performed.

  Cluster Update
  -  400 -- Cluster is in deleting state. Async operation is in progress . Cluster must complete its operation before any update operation is performed.
  -  400 -- KeyVaultProperties is not empty but has a bad format. See [key identifier update](../platform/customer-managed-keys.md#update-cluster-with-key-identifier-details).
  -  400 -- Failed to validate key in Key Vault. Could be due to lack of permissions or when key doesn’t exist. Verify that you [set key and access policy](../platform/customer-managed-keys.md#grant-key-vault-permissions) in Key Vault.
  -  400 -- Key is not recoverable. Key Vault must be set to Soft-delete and Purge-protection. See [Key Vault documentation](../../key-vault/general/soft-delete-overview.md)
  -  400 -- Operation cannot be executed now. Wait for the Async operation to complete and try again.
  -  400 -- Cluster is in deleting state. Wait for the Async operation to complete and try again.

  Cluster Get:
    -  404 -- Cluster not found, the cluster may have been deleted. If you try to create a cluster with that name and get conflict, the cluster is in soft-delete for 14 days. You can contact support to recover it, or use another name to create a new cluster. 

  Cluster Delete
    -  409 -- Can't delete a cluster while in provisioning state. Wait for the Async operation to complete and try again.

  Workspace link:
  -  404 -- Workspace not found. The workspace you specified doesn’t exist or was deleted.
  -  409 -- Workspace link or unlink operation in process.
  -  400 -- Cluster not found, the cluster you specified doesn’t exist or was deleted. If you try to create a cluster with that name and get conflict, the cluster is in soft-delete for 14 days. You can contact support to recover it.

  Workspace unlink:
  -  404 -- Workspace not found. The workspace you specified doesn’t exist or was deleted.
  -  409 -- Workspace link or unlink operation in process.

## Next steps

- Learn about [Log Analytics dedicated cluster billing](../platform/manage-cost-storage.md#log-analytics-dedicated-clusters)
- Learn about [proper design of Log Analytics workspaces](../platform/design-logs-deployment.md)
