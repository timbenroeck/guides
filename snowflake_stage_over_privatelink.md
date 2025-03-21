# Azure Private Link Integration with Snowflake

This document provides detailed instructions for configuring **Azure Private Link connectivity** between your Snowflake instance and Azure Blob Storage or Azure Data Lake Storage (ADLS) for Snowflake **Storage Integrations** and **External Stages**.

> **Important:** This feature requires a [Business Critical edition](https://docs.snowflake.com/en/user-guide/data-load-azure-private#label-azure-private-prereqs) Snowflake account. It is not available in government regions.

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

### Additional Resources

- [Azure Private Link Overview](https://docs.snowflake.com/en/user-guide/private-manage-endpoints-azure)
- [Azure Private Link for External Stages](https://docs.snowflake.com/en/user-guide/data-load-azure-private)
- [SYSTEM$PROVISION_PRIVATELINK_ENDPOINT](https://docs.snowflake.com/en/sql-reference/functions/system_provision_privatelink_endpoint)
- [CREATE STORAGE INTEGRATION](https://docs.snowflake.com/en/sql-reference/sql/create-storage-integration)

---

## Creating Private Link Endpoints in Snowflake

### Connecting Directly to Azure Storage Account (ADLS or Blob Storage)

To configure Private Link connectivity, you will need both the **Storage Account Resource ID** and the **Blob Endpoint FQDN**.

- **Option 1: Azure CLI**

  Use the following command to retrieve both values:

  ```bash
  az storage account show -g "<resource-group>" -n "<storage-account-name>" --query "[id, primaryEndpoints.blob]"
  ```

- **Option 2: Azure Portal**

  - Navigate to the storage account in the Azure Portal
  - In the left-hand menu, go to **Settings > Endpoints**
    - Copy the **Blob service endpoint**
  - Then go to **Properties** (or Overview in some regions)
    - Copy the **Resource ID**

  These values correspond to:
  - **Blob Endpoint FQDN** → `<storage-account-name>.blob.core.windows.net`
  - **Storage Account Resource ID** → `/subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Microsoft.Storage/storageAccounts/<storage-account-name>`

---

### Connecting via Private Link Service

Use this method if you're connecting to Azure Blob Storage through a Private Link Service (PLS). You will need the **PLS Resource ID** and the **Blob Endpoint FQDN**.

- **Option 1: Azure CLI**

  Use the following command to get the resource ID (replace with your values as needed):

  ```bash
  az network private-link-service show --resource-group "<resource-group>" --name "<pls-name>" --query "id"
  ```

- **Option 2: Azure Portal**

  - Navigate to the **Private Link Center** in the Azure Portal
  - Select **Private link services**
  - Click your Private Link Service name (`<pls-name>`)
  - Go to **Settings > Properties**
  - In the **Essentials** section, copy the **Resource ID**

  This value will look like:
  ```
  /subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Microsoft.Network/privateLinkServices/<pls-name>
  ```

Use the resource ID and endpoint FQDN to provision the endpoint in Snowflake:

```sql
SET pls_id = '/subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Microsoft.Network/privateLinkServices/<pls-name>';
SET adls_blob_fqdn = '<storage-account-name>.blob.core.windows.net';

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

Absolutely — here's a revised version of that section that:

- Clarifies the importance of the `USE_PRIVATELINK_ENDPOINT` flag
- Describes how to retrieve the **Azure Tenant ID** via CLI or Azure Portal
- Maintains your preferred formatting (bulleted items, no numbered steps)

---

## Create Storage Integration in Snowflake

Create a storage integration object in Snowflake to securely connect to your Azure Blob Storage or ADLS account using Private Link.

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

Create the integration using the following SQL command:

```sql
CREATE OR REPLACE STORAGE INTEGRATION <storage_integration_name>
  TYPE = EXTERNAL_STAGE
  STORAGE_PROVIDER = AZURE
  AZURE_TENANT_ID = '<azure-tenant-id>'
  STORAGE_ALLOWED_LOCATIONS = ('azure://<storage-account-name>.blob.core.windows.net/<container>/<path>/')
  USE_PRIVATELINK_ENDPOINT = TRUE
  ENABLED = TRUE;
```

[Docs: CREATE STORAGE INTEGRATION](https://docs.snowflake.com/en/sql-reference/sql/create-storage-integration)

- Once created, Snowflake will use private connectivity for all stages referencing this integration.
- The stage inherits the private connectivity configuration from the integration, so you do **not** need to specify `USE_PRIVATELINK_ENDPOINT` on the stage itself.
- You cannot mix public and private connectivity within a single storage integration.

---

## Azure Consent and IAM Role Assignment

### Grant Consent and Identify the Snowflake Service Principal

**Grant Consent via AZURE_CONSENT_URL**

- Run the following command in Snowflake to retrieve consent details:

```sql
DESC STORAGE INTEGRATION <storage_integration_name>;
```

- Locate the `AZURE_CONSENT_URL` in the result, and open it in a browser.

  **Some organizations require administrator approval in Microsoft Entra ID (formerly Azure AD) before granting consent. If you encounter a message that admin approval is needed, contact your Entra ID administration team.**

- This action registers the Snowflake Enterprise Application (service principal) in your Azure tenant. The application is added with no default permissions and must be granted access in the next step.

**Locate the Snowflake Service Principal Name**

- Note the value in `AZURE_MULTI_TENANT_APP_NAME` from the same `DESC STORAGE INTEGRATION` output.
- Use the prefix **before** the underscore to search in the Azure Portal.

For example, if `AZURE_MULTI_TENANT_APP_NAME` is `a1b2c3snowflakepacint_1234567891011`, search for `a1b2c3snowflakepacint`.

---

### Retrieve Client ID for Role Assignment

```sql
DESC STORAGE INTEGRATION <storage_integration_name>;

SET client_id = (
  SELECT REGEXP_SUBSTR($3, 'client_id=([^&]+)', 1, 1, 'e', 1)
  FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()))
  WHERE $1 = 'AZURE_CONSENT_URL'
);

SET adls_id = '/subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Microsoft.Storage/storageAccounts/<storage-account-name>';

SELECT CONCAT('az role assignment create --role "Storage Blob Data Contributor" --assignee "', $client_id, '" --scope "', $adls_id, '"' ) AS cli_command;
```

---

### Assign IAM Role using the Azure CLI

- Use the generated command from above to assign the role:

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
  - Search using the prefix of `AZURE_MULTI_TENANT_APP_NAME` (e.g., `a1b2c3snowflakepacint`)
  - Select the matching service principal
- Click **Review + assign**

You can verify the assignment by returning to the **Role Assignments** tab and confirming that the Snowflake service principal appears under the assigned role.

---

## Create and Test External Stage

```sql
CREATE OR REPLACE STAGE <stage_name>
  STORAGE_INTEGRATION = <storage_integration_name>
  URL = 'azure://<storage-account-name>.blob.core.windows.net/<container>/<path>/';
```

### Validate Access

- **List Files:**

```sql
LIST @<stage_name>;
```

- **Write Test Data:**

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

- **Read Test Data:**

```sql
SELECT $1, $2, $3, $4
FROM @<stage_name>/test-writing/data_0_0_0.csv.gz;
```

---

## Cost and Monitoring

- Endpoint provisioning and data transfers via Private Link are billed separately.
- Monitor usage using these views:

  - `ACCOUNT_USAGE.OUTBOUND_PRIVATELINK_ENDPOINT`
  - `ACCOUNT_USAGE.OUTBOUND_PRIVATELINK_DATA_PROCESSED`

[Docs: Cost Monitoring](https://docs.snowflake.com/en/user-guide/data-load-azure-private#outbound-private-connectivity-costs)

---

## Additional Resources
For support, contact your Snowflake account representative or support team.


**Documentation Links:**
- [Azure Private Link Overview](https://docs.snowflake.com/en/user-guide/private-manage-endpoints-azure)
- [Azure Private Link for External Stages](https://docs.snowflake.com/en/user-guide/data-load-azure-private)
- [SYSTEM$PROVISION_PRIVATELINK_ENDPOINT](https://docs.snowflake.com/en/sql-reference/functions/system_provision_privatelink_endpoint)
- [CREATE STORAGE INTEGRATION](https://docs.snowflake.com/en/sql-reference/sql/create-storage-integration)
- [Snowpipe Automation via Azure](https://docs.snowflake.com/en/user-guide/data-load-azure-snowpipe-auto)
