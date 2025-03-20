# Azure Private Link Integration with Snowflake - Setup Instructions

This document provides detailed instructions for configuring Azure Private Link connectivity between your Snowflake instance and Azure Blob Storage or Azure Data Lake Storage (ADLS).

## Prerequisites
- **Azure Subscription**: Ensure you have sufficient permissions to create and manage resources such as Private Link endpoints, storage accounts, and role assignments.
- **Snowflake Account**: Ensure you have the necessary privileges to configure integrations and execute administrative SQL statements.
- **Azure CLI**: Install the [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli) for convenience when running certain commands.

## 1. Creating Private Link Endpoints in Snowflake

### Connecting directly to Azure Storage Account (ADLS or Blob Storage)
Use this approach for direct private endpoint connectivity to an Azure Storage account. Obtain the Storage Account Resource ID via Azure Portal or Azure CLI:

```bash
az storage account show -g "<resource-group>" -n "<storage-account-name>" --query "id" -o tsv
```

Then provision the endpoint in Snowflake:

```sql
SELECT SYSTEM$PROVISION_PRIVATELINK_ENDPOINT(
    '/subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Microsoft.Storage/storageAccounts/<storage-account-name>', -- Storage Account Resource ID
    '<storage-account-name>.blob.core.windows.net', -- Storage Account Blob endpoint FQDN
    'blob' -- Resource type (blob)
);
```

### Connecting via Private Link Service
Use this method if you're connecting Snowflake to Azure Blob Storage through an Azure Private Link Service.

Replace placeholders with your Azure Private Link Service details:

```sql
SELECT SYSTEM$PROVISION_PRIVATELINK_ENDPOINT(
    '/subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Microsoft.Network/privateLinkServices/<private-link-service-name>', -- Private Link Service Resource ID
    '<storage-account-name>.blob.core.windows.net' -- Azure Storage Blob endpoint FQDN
);
```

### Confirm Endpoint Creation
Confirm that the private endpoint requests have been initiated and are in "Pending" status:

```sql
SELECT SYSTEM$GET_PRIVATELINK_ENDPOINTS_INFO();
```

---

## 2. Approve Private Endpoint Connections in Azure
Azure requires explicit approval of Private Endpoint connections.

- For **Private Link Service**:
  - Navigate to **Azure Portal > Private Link Center > Private Link Services**.
  - Select your service (**private-link-service-name**) > Settings > Private endpoint connections.
  - Approve the pending connection.

- For **Direct to ADLS/Blob Storage**:
  - Navigate to your **Storage Account > Security + Networking > Networking**.
  - Go to **Private Endpoint Connections** and approve the pending request.

---

## 3. Creating a Storage Integration in Snowflake
After establishing connectivity, set up a Storage Integration object to interact securely with Azure storage using private endpoints.

```sql
CREATE OR REPLACE STORAGE INTEGRATION <storage_integration_name>
  TYPE = EXTERNAL_STAGE
  STORAGE_PROVIDER = AZURE
  AZURE_TENANT_ID = '<azure-tenant-id>'
  STORAGE_ALLOWED_LOCATIONS = ('azure://<storage-account-name>.blob.core.windows.net/<path>/')
  USE_PRIVATELINK_ENDPOINT = TRUE
  ENABLED = TRUE;
```

---

## 4. Azure Consent and IAM Role Assignment

### Extract Consent URL and Service Principal details:
Retrieve important details needed for Azure authentication and role assignment from Snowflake:

```sql
DESC STORAGE INTEGRATION <storage_integration_name>;
```

From the output:
- Navigate to the URL specified in `AZURE_CONSENT_URL` to authorize the Snowflake Service Principal.
- Take note of the `AZURE_MULTI_TENANT_APP_NAME` (Service Principal name, without suffix).

### Retrieve Service Principal Client ID
Run these SQL commands to extract the client ID for role assignments:

```sql
DESC STORAGE INTEGRATION <storage_integration_name>;

SET client_id=(
  SELECT REGEXP_SUBSTR($3, 'client_id=([^&]+)', 1, 1, 'e', 1)
  FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()))
  WHERE $1 = 'AZURE_CONSENT_URL'
);

SET storage_account_name = '<storage-account-name>';
SET storage_resource_group = '<resource-group>';

SELECT
  CONCAT(
    'az role assignment create --role "Storage Blob Data Contributor" --assignee "',
    $client_id,
    '" --scope "$(az storage account show -g "',
    $storage_resource_group,
    '" -n "',
    $storage_account_name,
    '" --query "id" -o tsv)"'
  ) AS cli_command;
```

### Assign IAM Role
Execute the generated `cli_command` using Azure CLI or assign the role via Azure Portal:
- **Azure Portal**: Storage Account > Access Control (IAM) > Add Role Assignment > "Storage Blob Data Contributor".

---

## 5. Create and Test Stage in Snowflake
Define an external stage referencing your storage integration:

```sql
CREATE OR REPLACE STAGE <stage_name>
  STORAGE_INTEGRATION = <storage_integration_name>
  URL = 'azure://<storage-account-name>.blob.core.windows.net/<path>';
```

### Validate Connectivity and Permissions:

- **Test Listing Files**:
```sql
LIST @<stage_name>;
```

- **Test Writing Data**:
```sql
COPY INTO @<stage_name>/test-writing/
FROM (
    SELECT
        SEQ8() AS id,
        RANDOM() AS random_value,
        CURRENT_TIMESTAMP AS generated_at,
        'SampleData_' || SEQ8() AS description
    FROM TABLE(GENERATOR(ROWCOUNT => 1000))
);
```

- **Test Reading Data**:
```sql
SELECT $1, $2, $3, $4
FROM @<stage_name>/test-writing/data_0_0_0.csv.gz;
```

---

## Additional Documentation and Resources
- [Snowflake Documentation – Azure Private Link](https://docs.snowflake.com/en/user-guide/private-manage-endpoints-azure)
- [Snowflake Documentation – Loading Data Using Azure Private Link](https://docs.snowflake.com/en/user-guide/data-load-azure-private)

---

**Important:**
Replace all placeholders (e.g., `<subscription-id>`, `<resource-group>`, `<storage-account-name>`, `<storage_integration_name>`, `<stage_name>`, etc.) with the appropriate values specific to your Azure environment and Snowflake account configuration.

If you encounter any issues or have questions, please reach out to your technical account representative or support team.

---
