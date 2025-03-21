# Private Link Integration for Snowflake Egress into Azure Data Lake Store

This document provides detailed instructions for configuring **Azure Private Link connectivity** from Snowflake to Azure Data Lake Store (ADLS) via:

- **Storage Integrations (External Stages and External Tables)**
- **External Volumes (Iceberg Tables)**

---

## When to use Storage Integrations vs. External Volumes?

It's important to select the correct integration method based on your specific use case:

- **Storage Integrations (External Stages & External Tables)**
  - Used to access and query external data directly from Azure Blob Storage (ADLS).
  - Supports reading from file formats: JSON, Avro, ORC, XML, Parquet, Delta.
  - This method is ideal when you need flexible schema-on-read capabilities for semi-structured or structured data stored externally.
  - Historically, Parquet and Delta tables have been queried via External Tables, but Snowflake recommends migrating to Iceberg Tables via External Volumes for better performance, ACID properties, and metadata management.

- **External Volumes (Iceberg Tables)**
  - Used exclusively for reading data stored in Apache Iceberg format, including Iceberg-managed tables built on Parquet or Delta files.
  - Provides improved performance through efficient metadata handling, transactional consistency (ACID compliance), and robust schema evolution.
  - Preferred for new implementations involving Parquet and Delta files, as Iceberg Tables via External Volumes are actively replacing External Tables for these formats.
  - Suitable for multi-engine environments (Snowflake, Spark, Hive, Presto), facilitating unified data management across diverse analytical workloads.

---

## Prerequisites

- **Azure Subscription**: Ensure you have sufficient permissions to create and manage resources such as Private Link endpoints, storage accounts, and role assignments.
- **Snowflake Account**: Use the `ACCOUNTADMIN` role to configure integrations and provision private endpoints.
- **Azure CLI**: Install the [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli) for convenience when running certain commands.

### Replace All Placeholders

Ensure the following placeholders are replaced:

- `<subscription-id>`
- `<resource-group>`
- `<storage-account-name>`
- `<storage_integration_name>`
- `<stage_name>`
- `<container>`
- `<path>`
- `<azure-tenant-id>`
- `<external_volume_name>`
- `<iceberg_table_name>`

### Documentation Resources

- [Azure Private Link Overview](https://docs.snowflake.com/en/user-guide/private-manage-endpoints-azure)
- [Azure Private Link for External Volumes](https://docs.snowflake.com/en/user-guide/tables-iceberg-configure-external-volume-azure-private)
- [Azure Private Link for External Stages](https://docs.snowflake.com/en/user-guide/data-load-azure-private)
- [SYSTEM$PROVISION_PRIVATELINK_ENDPOINT](https://docs.snowflake.com/en/sql-reference/functions/system_provision_privatelink_endpoint)
- [CREATE EXTERNAL VOLUME](https://docs.snowflake.com/en/sql-reference/sql/create-external-volume)

---

## Creating Private Link Endpoints in Snowflake

Private Link endpoints in Snowflake are used to route traffic securely to ADLS over a private network. These endpoints are tightly bound to a hostname and cannot be modified after they are created.

#### Hostname Options

The hostname determines which traffic routes through the Private Link:

- A **wildcard hostname** (`*.blob.core.windows.net`) enables access to multiple storage accounts using one endpoint, but only for account that have a storage integration or external volume with the `USE_PRIVATELINK_ENDPOINT` parameter enabled
- A **specific hostname** only allows traffic to that one storage account
- If you create both wildcard and specific endpoints, Snowflake will prioritize the one that matches the exact hostname

 - You can **only have one hostname per endpoint**
 - You **cannot associate multiple hostnames** to the same Private Link connection
 - Deprovisioning triggers a **7-day soft-delete period** where the endpoint remains queued for deletion and cannot be recreated with the same values
 - If you attempt to re-provision during this time, you will receive an error similar to:
   `"Private endpoint for resource ... has been queued for deletion..."`
   Use `SYSTEM$RESTORE_PRIVATELINK_ENDPOINT` if appropriate, or contact support to expedite removal.

> **Important:**
> As of March 2025, the hostname (FQDN) you specify is final and cannot be updated later.
> If you need to change it (e.g., you mistyped it, need to switch from a wildcard to a specific storage account or vice versa), you must deprovision the endpoint and **contact Snowflake Support** to force delete it to bypass the 7-day soft-delete period.

---

### Connecting Directly to Azure Storage Account (ADLS or Blob Storage)

This option creates a private endpoint from Snowflake directly to a storage account (recommended). You must include the required **subresource type**, which for Blob storage is `'blob'`.

> **Subresource note:**
> This argument is required when targeting Azure Storage directly.

#### Required Values

- **Storage Account Resource ID**
- **Blob Endpoint FQDN** (e.g., `<storage-account-name>.blob.core.windows.net`)

- **Option 1: Azure CLI**

  ```bash
  az storage account show -g "<resource-group>" -n "<storage-account-name>" --query "[id, primaryEndpoints.blob]"
  ```

- **Option 2: Azure Portal**

  - Go to your storage account
  - **Settings > Endpoints**: copy the **Blob service endpoint**
  - **Properties (or Overview)**: copy the **Resource ID**

#### Provision the Endpoint in Snowflake

```sql
USE ROLE ACCOUNTADMIN;

SET adls_id = '/subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Microsoft.Storage/storageAccounts/<storage-account-name>';
SET adls_blob_fqdn = '<storage-account-name>.blob.core.windows.net';

SELECT SYSTEM$PROVISION_PRIVATELINK_ENDPOINT($adls_id, $adls_blob_fqdn, 'blob');
```

---

### Connecting via Private Link Service (PLS)

Use this method if your architecture uses an Azure Private Link Service as a proxy or relay to Blob Storage. This is common in advanced networking scenarios or when exposing shared services.

#### Get the Private Link Service Resource ID

- **Option 1: Azure CLI**

  ```bash
  az network private-link-service show \
    --resource-group "<resource-group>" \
    --name "<pls-name>" \
    --query "id"
  ```

- **Option 2: Azure Portal**

  - Navigate to the **Private Link Center**
  - Select **Private link services**
  - Choose your service (`<pls-name>`)
  - Go to **Settings > Properties**
  - Copy the **Resource ID** under **Essentials**

#### Provision the Endpoint in Snowflake

```sql
SET pls_id = '/subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Microsoft.Network/privateLinkServices/<pls-name>';

-- Option 1: Specific hostname
SET adls_blob_fqdn = '<storage-account-name>.blob.core.windows.net';

-- Option 2: Wildcard hostname
-- SET adls_blob_fqdn = '*.blob.core.windows.net';

SELECT SYSTEM$PROVISION_PRIVATELINK_ENDPOINT($pls_id, $adls_blob_fqdn);
```

---

### Confirm Endpoint Status

After initiating the private endpoint connection, use the following command to check its status:

```sql
SELECT SYSTEM$GET_PRIVATELINK_ENDPOINTS_INFO();
```

If the result includes `"status": "PENDING"`, the connection must be approved in Azure before Snowflake can use it.

Your **cloud engineering or networking team** may need to approve this request, depending on your organization's Azure governance model.

Once the approval has been completed, re-run the command above to confirm that the status has changed to `"APPROVED"`.

[Docs: Endpoint Status](https://docs.snowflake.com/en/user-guide/data-load-azure-private#step-2-approve-the-private-endpoint-connection)

## Approve Private Endpoint in Azure

Use the Azure Portal to approve the pending private endpoint request.

- **Private Link Service**:
  Azure Portal → Private Link Center → Private Link Services → [your service] → Private Endpoint Connections → Approve

- **Direct to Storage Account**:
  Azure Portal → Storage Account → Networking → Private Endpoint Connections → Approve

Approval may require action from your organization's **network or cloud infrastructure team**, especially if you do not have owner-level permissions on the target resource.

---

## Cost and Monitoring

- Endpoint provisioning and data transfers via Private Link are billed separately.
- Monitor usage using these views:

  - `ACCOUNT_USAGE.OUTBOUND_PRIVATELINK_ENDPOINT`
  - `ACCOUNT_USAGE.OUTBOUND_PRIVATELINK_DATA_PROCESSED`

[Docs: Cost Monitoring](https://docs.snowflake.com/en/user-guide/data-load-azure-private#outbound-private-connectivity-costs)

---

## Connect to ADLS over Private Link
You a [Storage Integration](https://docs.snowflake.com/en/sql-reference/sql/create-storage-integration) or [External Volume](https://docs.snowflake.com/en/user-guide/tables-iceberg-configure-external-volume-azure) object in Snowflake to securely connect to your Azure Blob Storage (ADLS) account using Private Link.

The `USE_PRIVATELINK_ENDPOINT = TRUE` property is optional in the syntax, but **required** when using Azure Private Link. This instructs Snowflake to route traffic through the provisioned private endpoint instead of the public internet.

Before creating the integration, gather your **Azure Tenant ID**. This identifies the Azure Active Directory (Entra ID) tenant that owns the storage account.

You can retrieve the tenant ID in either of the following ways:

- **Azure CLI**:

  ```bash
  az account show --query "[tenantId]"
  ```

- **Azure Portal**:
  - Go to the [Microsoft Entra ID overview page](https://portal.azure.com/#view/Microsoft_AAD_IAM/ActiveDirectoryMenuBlade/~/Overview)
  - Under **Basic Information**, copy the **Tenant ID** field

### Create the Storage Integration using the following SQL command:

```sql
CREATE OR REPLACE STORAGE INTEGRATION <storage_integration_name>
  TYPE = EXTERNAL_STAGE
  STORAGE_PROVIDER = AZURE
  AZURE_TENANT_ID = '<azure-tenant-id>'
  STORAGE_ALLOWED_LOCATIONS = ('azure://<storage-account-name>.blob.core.windows.net/<container>/<path>/')
  USE_PRIVATELINK_ENDPOINT = TRUE
  ENABLED = TRUE;
```

Once created, Snowflake will use private connectivity for all [Azure External Stages](https://docs.snowflake.com/en/user-guide/data-load-azure-create-stage) referencing this integration.

### Create an External Volume that uses Private Link

```sql
CREATE OR REPLACE EXTERNAL VOLUME <external_volume_name>
  STORAGE_LOCATIONS = (
    (
      NAME = '<external_volume_name>'
      STORAGE_PROVIDER = 'AZURE'
      STORAGE_BASE_URL = 'azure://<storage-account-name>.blob.core.windows.net/<container>/'
      AZURE_TENANT_ID = '<azure-tenant-id>'
      USE_PRIVATELINK_ENDPOINT = TRUE
    )
  );
```

Once created, Snowflake will use private connectivity for all [Iceberg Tables](https://docs.snowflake.com/en/user-guide/tables-iceberg-create) referencing this external volume.

---

## Azure Consent and IAM Role Assignment

### Grant Consent and Identify the Snowflake Service Principal

**Grant Consent via AZURE_CONSENT_URL**

- Run the following command in Snowflake to retrieve consent details:

```sql
DESC STORAGE INTEGRATION <storage_integration_name>;
-- or --
DESC EXTERNAL VOLUME <external_volume_name>;
```

- Locate the `AZURE_CONSENT_URL` in the result, and open it in a browser.

  **Some organizations require administrator approval in Microsoft Entra ID (formerly Azure AD) before granting consent. If you encounter a message that admin approval is needed, contact your Entra ID administration team.**

- This action registers the Snowflake Enterprise Application (service principal) in your Azure tenant. The application is added with no default permissions and must be granted access in the next step.

**Locate the Snowflake Service Principal Name**

- Note the value in `AZURE_MULTI_TENANT_APP_NAME` from the `DESC STORAGE INTEGRATION` or `DESC EXTERNAL VOLUME` output.
- Use the prefix **before** the underscore to search in the Azure Portal.

For example, if `AZURE_MULTI_TENANT_APP_NAME` is `a1b2c3snowflakepacint_1234567891011`, search for `a1b2c3snowflakepacint`.

---

### Assign IAM Role using the Azure CLI

```sql
DESC STORAGE INTEGRATION <storage_integration_name>;
-- or --
DESC EXTERNAL VOLUME <external_volume_name>;

SELECT cli_command
FROM
(
    SELECT "property_value" AS extract_client_id,
        REGEXP_SUBSTR(extract_client_id, 'client_id=([^&]+)', 1, 1, 'e', 1) AS client_id,
        CONCAT('az role assignment create --role "Storage Blob Data Contributor" --assignee "', client_id, '" --scope "', $adls_id, '"' ) AS cli_command
    FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()))
    WHERE contains("property_value", 'client_id=')
)
;
```

Use the generated command from above to assign the role, example:

```bash
az role assignment create --role "Storage Blob Data Contributor" --assignee "<client-id>" --scope "/subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Microsoft.Storage/storageAccounts/<storage-account-name>"
```

[Azure Role Assignment CLI Reference](https://learn.microsoft.com/en-us/cli/azure/role/assignment)

---

### Assign IAM Role via Azure Portal

- Navigate to [https://portal.azure.com](https://portal.azure.com) and open your **Storage Account**
- In the left-hand menu, click **Access Control (IAM)**
- Click **+ Add > Add Role Assignment**
- In the wizard:
  - Search for **Storage Blob Data Contributor**
  - Click **Next**
- In the **Members** tab:
  - Choose **User, group, or service principal**
  - Click **Select members**
  - Search using the prefix of `AZURE_MULTI_TENANT_APP_NAME` (e.g., `a1b2c3snowflakepacint` in `a1b2c3snowflakepacint_1234567891011`)
  - Select the matching service principal
- Click **Review + assign**

You can verify the assignment by returning to the **Role Assignments** tab and confirming that the Snowflake service principal appears under the assigned role.

---

## Test External Volumes by creating an Iceberg Table

```sql
CREATE OR REPLACE ICEBERG TABLE <iceberg_table_name>
  CATALOG = 'SNOWFLAKE'
  EXTERNAL_VOLUME = '<external_volume_name>'
  BASE_LOCATION = '<optional-relative-path-in-container>/<iceberg_table_name>'
  AS
  SELECT
    SEQ8() AS id,
    RANDOM() AS random_value,
  FROM TABLE(GENERATOR(ROWCOUNT => 1000));
```

### Test Reading from the Table

```sql
SELECT * FROM <iceberg_table_name> LIMIT 10;
```

---

## Test Storage Integrations by creating an External Stage

```sql
CREATE OR REPLACE STAGE <stage_name>
  STORAGE_INTEGRATION = <storage_integration_name>
  URL = 'azure://<storage-account-name>.blob.core.windows.net/<container>/<path>/';
```

**Test listing files:**

```sql
LIST @<stage_name>;
```

**Test writing data:**

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

**Test reading data:**

```sql
SELECT $1, $2, $3, $4
FROM @<stage_name>/test-writing/data_0_0_0.csv.gz;
```
