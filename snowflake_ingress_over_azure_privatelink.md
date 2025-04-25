# Setup Private Link for Snowflake Ingress from Azure

This document outlines how to securely configure Snowflake using **Azure Private Link**, shared **DNS infrastructure**, and **Snowflake network policies**. The approach standardizes Private Link access per region while enabling individual account-level enforcement of private-only traffic.

---

### Phase 1: Set Up Azure Private Link and Private DNS

#### Objective

Establish **one Azure Private Endpoint per region** to connect to Snowflake’s regional Private Link service. All Snowflake accounts in that region will share this endpoint. This design reduces overhead while maintaining secure access boundaries.

---

#### Steps

1. **Create the Azure Private Endpoint**

    - Use the `privatelink-pls-id` returned by the following Snowflake command:

        ```sql
        SELECT SYSTEM$GET_PRIVATELINK_CONFIG();
        ```

    - This function returns a JSON object containing:

        - The **Private Link Service ID** (`privatelink-pls-id`)

        - Required **DNS hostnames** for the account

        - Additional services (e.g., Snowsight, registry, OCSP)


    Example output (anonymized for general use):

    ```json
    {
      "regionless-snowsight-privatelink-url": "app-org-abc.privatelink.snowflakecomputing.com",
      "privatelink-account-name": "xx00000.west-us-2.privatelink",
      "snowsight-privatelink-url": "app.west-us-2.privatelink.snowflakecomputing.com",
      "regionless-privatelink-ocsp-url": "ocsp.org-abc.privatelink.snowflakecomputing.com",
      "privatelink-account-url": "xx00000.west-us-2.privatelink.snowflakecomputing.com",
      "privatelink-pls-id": "sf-pvlinksvc-azwestus2.<guid>.westus2.azure.privatelinkservice",
      "spcs-registry-privatelink-url": "org-abc.registry.privatelink.snowflakecomputing.com",
      "app-service-privatelink-url": "*.eatzyx.privatelink.snowflake.app",
      "regionless-privatelink-account-url": "org-abc.privatelink.snowflakecomputing.com",
      "spcs-auth-privatelink-url": "sfc-endpoint-login.eatzyx.privatelink.snowflakecomputing.com",
      "privatelink_ocsp-url": "ocsp.xx00000.west-us-2.privatelink.snowflakecomputing.com"
    }
    ```

2. **Capture the Private Endpoint IP Address** Example: `10.2.0.5`. This will be used for all DNS mappings in the region.

3. **Create Two Azure Private DNS Zones**

    - `privatelink.snowflakecomputing.com`

    - `privatelink.snowflake.app`

4. **Create DNS Records (A Records)**

    Example entries in the respective zones:

    **In `privatelink.snowflakecomputing.com`:**

    ```
    *.west-us-2                         A   10.2.0.5
    *.org-abc                           A   10.2.0.5
    org-abc                             A   10.2.0.5
    app-org-abc                         A   10.2.0.5
    ocsp.org-abc                        A   10.2.0.5
    org-abc.registry                    A   10.2.0.5
    ```

    **In `privatelink.snowflake.app`:**

    ```
    *.eatzyx                            A   10.2.0.5
    sfc-endpoint-login.eatzyx          A   10.2.0.5
    ```

    These entries cover:

    - Snowflake UI (Snowsight)

    - Snowflake's internal registry/auth services

    - OCSP endpoints for certificate validation

    - All connection strings for Snowflake accounts in the `west-us-2` region

5. **Wildcard Record Usage**

    - Azure Private DNS now supports wildcard A records.

    - This allows broad matching across accounts or services without creating individual records for each one.

    - Examples:

        - `*.west-us-2.privatelink.snowflakecomputing.com`

        - `*.org-abc.privatelink.snowflakecomputing.com`

        - `*.eatzyx.privatelink.snowflake.app`

6. **Link the DNS Zones to VNets**

    - Link both zones to all VNets hosting workloads that access Snowflake.


---

### Phase 2: Apply Snowflake Network Policies (Start with User-Level Testing)

Once DNS and Private Link are configured, the next step is to **enforce private-only access** to Snowflake. It’s strongly recommended to begin by applying the policy to a single user, verify functionality, then expand it account-wide.

#### 1. Retrieve Authorized Endpoint(s)

Run the following to identify the Snowflake-internal identifier (`linkIdentifier`) of the authorized Azure Private Endpoint:

```sql
SELECT SYSTEM$GET_PRIVATELINK_AUTHORIZED_ENDPOINTS();
```

Example result:

```json
[
  {
    "endpointId": ".../privateEndpoints/endpoint-snowflake-westus2",
    "linkIdentifier": "637558822"
  }
]
```

#### 2. Create Network Rules

```sql
CREATE NETWORK RULE IF NOT EXISTS ALLOW_PL_WESTUS2
  TYPE = AZURELINKID
  VALUE_LIST = ('0123456789');

CREATE NETWORK RULE IF NOT EXISTS BLOCK_ALL_IPV4
  TYPE = IPV4
  MODE = INGRESS
  VALUE_LIST = ('0.0.0.0/0');
```

#### 3. Create and Apply a Network Policy for a Specific User

```sql
CREATE NETWORK POLICY IF NOT EXISTS USER_POLICY_JSMITH
  ALLOWED_IP_LIST = ('111.222.33.44', '44.33.22.11')
  ALLOWED_NETWORK_RULE_LIST = ('ALLOW_PL_WESTUS2')
  BLOCKED_NETWORK_RULE_LIST = ('BLOCK_ALL_IPV4')
  COMMENT = 'Restrict JSMITH to Private Link traffic and specific IPs';
```

Apply it to the user:

```sql
ALTER USER JSMITH SET NETWORK_POLICY = USER_POLICY_JSMITH;
```

> This setup ensures the user `JSMITH` can only access Snowflake via the approved Private Link endpoint or only from the specified IP addresses. All public internet traffic is explicitly blocked.

---

### Optional: Expand to Account-Level Enforcement

Once verified at the user level, the same structure can be promoted to an account-wide policy.

```sql
CREATE NETWORK POLICY IF NOT EXISTS POLICY_PRIVATE_ONLY
  ALLOWED_NETWORK_RULE_LIST = ('ALLOW_PL_WESTUS2')
  BLOCKED_NETWORK_RULE_LIST = ('BLOCK_ALL_IPV4')
  COMMENT = 'Restrict account access to approved Private Link only';
```

Apply it:

```sql
ALTER ACCOUNT SET NETWORK_POLICY = POLICY_PRIVATE_ONLY;
```
