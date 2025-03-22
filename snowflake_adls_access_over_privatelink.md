# Private Link Integration for Snowflake Egress into Azure Data Lake Store

This document provides detailed instructions for configuring **Azure Private Link connectivity** for Snowflake to Azure Data Lake Store (ADLS) connectivity.  It will provide instructions for creating **Storage Integrations** used byExternal Stages and External Tables, as well as, **External Volumes** which are required for reading and writing Apache Iceberg tables.

---

## When to use Storage Integrations vs. External Volumes?

It's important to select the correct integration method based on your specific use case:

- **Storage Integrations**:
  - Used to access and query external data directly from Azure Blob Storage (ADLS) via External Stages and/or External Tables.
  - Supports reading from file formats: JSON, Avro, ORC, XML, Parquet, Delta.
  - This method is ideal when you need flexible schema-on-read capabilities for semi-structured or structured data stored externally.
  - Historically, Parquet and Delta tables have been queried via External Tables, but Snowflake recommends migrating to Iceberg Tables via External Volumes for better performance, ACID properties, and metadata management.

- **External Volumes**:
  - Used exclusively for reading and writing data stored in Apache Iceberg format, including Iceberg-managed tables built on Parquet or Delta files.
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

Private Link endpoints in Snowflake are used to route traffic securely to ADLS over an Azure Private Link connection. These endpoints are tightly bound to a hostname and cannot be modified after they are created.

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

You will need the **Resource ID** and the **Blob Endpoint FQDN** for the Azure Data Lake Store (ADLS) account you are integrating. You can get them from the Azure Portal or by using the CLI:

  - Azure Portal

    - Navigate to to your storage account
    - **Settings > Endpoints**: copy the **Blob service endpoint**
    - **Properties (or Overview)**: copy the **Resource ID**

  - Azure CLI

    ```bash
    az storage account show -g "<resource-group>" -n "<storage-account-name>" --query "[id, primaryEndpoints.blob]"
    ```

#### Provision the Endpoint in Snowflake

```sql
USE ROLE ACCOUNTADMIN;

SET adls_id = '/subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Microsoft.Storage/storageAccounts/<storage-account-name>';
SET adls_blob_fqdn = '<storage-account-name>.blob.core.windows.net';

SELECT SYSTEM$PROVISION_PRIVATELINK_ENDPOINT($adls_id, $adls_blob_fqdn, 'blob');
```

After provisioning the private link endpoint, use the `SYSTEM$GET_PRIVATELINK_ENDPOINTS_INFO()` to check its status. If the endpoint status is `"status": "PENDING"`, the endpoint has been created and is ready for approval in Azure.

---

### Connecting via Private Link Service (PLS)

Use this method if your architecture uses an Azure Private Link Service as a proxy or relay to Blob Storage. You will need the **Resource ID** for the Azure Private Link Service you are integrating. You can get this from the Azure Portal or by using the CLI:

  - Azure Portal

    - Navigate to the **Private Link Center**
    - Select **Private link services**
    - Choose your service (`<pls-name>`)
    - Go to **Settings > Properties**
    - Copy the **Resource ID** under **Essentials**

  - Azure CLI

    ```bash
    az network private-link-service show \
      --resource-group "<resource-group>" \
      --name "<pls-name>" \
      --query "id"
    ```



#### Provision the Endpoint in Snowflake

```sql
SET pls_id = '/subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Microsoft.Network/privateLinkServices/<pls-name>';

-- Option 1: Specific hostname
SET adls_blob_fqdn = '<storage-account-name>.blob.core.windows.net';

-- Option 2: Wildcard hostname
-- SET adls_blob_fqdn = '*.blob.core.windows.net';

SELECT SYSTEM$PROVISION_PRIVATELINK_ENDPOINT($pls_id, $adls_blob_fqdn);
```

After provisioning the private link endpoint, use the `SYSTEM$GET_PRIVATELINK_ENDPOINTS_INFO()` to check its status. If the endpoint status is `"status": "PENDING"`, the endpoint has been created and is ready for approval in Azure.

---

## Approve the Endpoint Connection

Use the Azure Portal to approve the pending private endpoint request:

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
Once the `SYSTEM$GET_PRIVATELINK_ENDPOINTS_INFO()` function reports the stats as `"status": "APPROVED"` you can use a [Storage Integration](https://docs.snowflake.com/en/sql-reference/sql/create-storage-integration) or [External Volume](https://docs.snowflake.com/en/user-guide/tables-iceberg-configure-external-volume-azure) to securely connect to your Azure Blob Storage (ADLS) account using the private link endpoint.

This is done by setting the `USE_PRIVATELINK_ENDPOINT` property to `TRUE`.  This parameter is optional when creating the resources, but is **required** to route the traffic over the private endpoint.  Before creating an External Volume or Storage Integration, gather your **Azure Tenant ID**. This identifies the Azure Entra ID (Azure Active Directory) tenant that owns the storage account. You can get this from the Azure Portal or by using the CLI:

  - Azure Portal:
    - Go to the [Microsoft Entra ID overview page](https://portal.azure.com/#view/Microsoft_AAD_IAM/ActiveDirectoryMenuBlade/~/Overview)
    - Under **Basic Information**, copy the **Tenant ID** field
  - Azure CLI:

    ```bash
    az account show --query "[tenantId]"
    ```

### Create a Storage Integration using the following SQL command:

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

## Snowflake Authentication and Authorization

When connecting to Azure Data Lake Storage (ADLS) from Snowflake, an Azure Private Link Endpoint is used to ensure a secure and private networking connection. However, this only provides the networking layer. For full access, Snowflake must also be **authenticated and authorized** to interact with data in your ADLS account. This is achieved using an **Azure Entra ID (formerly Azure Active Directory) Enterprise Application (Service Principal).**

Snowflake uses an enterprise application registered in your Azure Entra ID tenant—as its **Service Principal identity** for authentication and authorization when accessing ADLS files.

---

### Register the Snowflake Enterprise Application

To register the Snowflake Enterprise Application in your tenant, run the `DESCRIBE` command on your Storage Integration or External Volume to retrieve the `AZURE_CONSENT_URL`. Navigating to this URL in a browser initiates the registration process.

```sql
DESCRIBE STORAGE INTEGRATION <storage_integration_name>;
-- or --
DESCRIBE EXTERNAL VOLUME <external_volume_name>;
```

Locate the `AZURE_CONSENT_URL` in the output and open it in a browser.

> **Note:** Some organizations require administrator approval in Microsoft Entra ID before granting consent. If you see a message requesting admin approval, contact your Entra ID admin team.

Once registered, the application has no default Access Control (IAM) roles—these must be assigned manually.

---

### Assign an Azure Access Control (IAM) Role

To authorize Snowflake’s access to ADLS, assign an IAM role (e.g., **Storage Blob Data Contributor**) to the Snowflake Service Principal. You’ll need either the **client ID** (available in the `AZURE_CONSENT_URL`) or the **Service Principal name** (found in `AZURE_MULTI_TENANT_APP_NAME`).

For example, if the value of `AZURE_MULTI_TENANT_APP_NAME` is `a1b2c3snowflakepacint_1234567891011`, the e **Service Principal name** is `a1b2c3snowflakepacint`.

#### Assign IAM Role via Azure Portal

1. Go to [https://portal.azure.com](https://portal.azure.com) and open your **Storage Account**
2. In the left menu, select **Access Control (IAM)**
3. Click **+ Add > Add role assignment**
4. In the role assignment wizard:
   - Search for and select **Storage Blob Data Contributor**
   - Click **Next**
5. Under **Members**:
   - Choose **User, group, or service principal**
   - Click **Select members**
   - Search using the prefix from `AZURE_MULTI_TENANT_APP_NAME`
   - Select the matching service principal
6. Click **Review + assign**

To confirm, return to the **Role Assignments** tab and verify that the Snowflake Service Principal appears under the assigned role.

---

#### Assign IAM Role via Azure CLI

Use the following Snowflake SQL to generate a CLI command that assigns the `Storage Blob Data Contributor` role:

```sql
DESC STORAGE INTEGRATION <storage_integration_name>;
-- or --
DESC EXTERNAL VOLUME <external_volume_name>;

SELECT cli_command
FROM (
    SELECT "property_value" AS extract_client_id,
           REGEXP_SUBSTR(extract_client_id, 'client_id=([^&]+)', 1, 1, 'e', 1) AS client_id,
           CONCAT('az role assignment create --role "Storage Blob Data Contributor" --assignee "', client_id, '" --scope "', $adls_id, '"') AS cli_command
    FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()))
    WHERE CONTAINS("property_value", 'client_id=')
);
```

Copy and run the generated cli_command:

```bash
az role assignment create --role "Storage Blob Data Contributor" --assignee "<client-id>" --scope "/subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Microsoft.Storage/storageAccounts/<storage-account-name>"
```

Refer to the [Azure Role Assignment CLI Reference](https://learn.microsoft.com/en-us/cli/azure/role/assignment) for more options.

---

## Validate Configuration

Once network access and authorization are correctly configured, you can verify functionality creating either an **Iceberg Table** (for External Volumes) or an **External Stage** (for Storage Integrations).

---

### Validate External Volumes with an Iceberg Table

```sql
CREATE OR REPLACE ICEBERG TABLE <iceberg_table_name>
  CATALOG = 'SNOWFLAKE'
  EXTERNAL_VOLUME = '<external_volume_name>'
  BASE_LOCATION = '<optional-relative-path-in-container>/<iceberg_table_name>'
  AS
  SELECT
    SEQ8() AS id,
    RANDOM() AS random_value
  FROM TABLE(GENERATOR(ROWCOUNT => 1000));
```

**Read sample data:**

```sql
SELECT * FROM <iceberg_table_name> LIMIT 10;
```

---

### Validate Storage Integrations with an External Stage

```sql
CREATE OR REPLACE STAGE <stage_name>
  STORAGE_INTEGRATION = <storage_integration_name>
  URL = 'azure://<storage-account-name>.blob.core.windows.net/<container>/<path>/';
```

**List files in the stage:**

```sql
LIST @<stage_name>;
```

**Write test data:**

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

**Read back test data:**

```sql
SELECT $1, $2, $3, $4
FROM @<stage_name>/test-writing/data_0_0_0.csv.gz;
```

---
