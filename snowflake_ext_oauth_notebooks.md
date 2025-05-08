# Secure OAuth Integration Between Microsoft Entra ID and Snowflake

###  With Token-Based Login from a Python Notebook

---

## Overview

This guide walks through configuring an **external OAuth security integration** between **Microsoft Entra ID (Entra ID)** and **Snowflake**, and using a **Databricks notebook** to authenticate via **OAuth 2.0** using **MSAL** and connect with the **Snowflake Python Connector**.

OAuth is preferred for enterprise authentication because it avoids hardcoded credentials and enables identity federation.

---

## Step 1: Register an Application in Entra ID

### Purpose:

This app registration represents your client application that will request access tokens from Entra ID to authenticate into Snowflake.

### How to:

In **Azure Portal**:

1. Go to **Microsoft Entra ID → App registrations → New registration**
2. Name: `snowflake_ext_oauth_client`
3. Under **Supported account types**, select:

   > *Accounts in this organizational directory only (Snowflake SE Sandbox only - Single tenant)*
   > This restricts access to users within your Entra ID tenant.
4. Leave Redirect URI blank for now
5. Click **Register**
6. Copy and store:

   * `<client-id>` = Application (client) ID
   * `<tenant-id>` = Directory (tenant) ID

---

## Step 2: Define a Custom OAuth Scope (“Expose an API”)

### Purpose:

Define a custom permission (scope) that clients can request. This will be embedded in the token and used by Snowflake to verify the token’s purpose.

### How to:

1. In the App Registration, go to **Expose an API**
2. Set **Application ID URI** to:
   `api://<client-id>`
   *(e.g., `api://8cb3c074-6a4b-4a07-9dff-3ae1b7ad0657`)*
3. Click **Add a scope**:

   * Scope name: `SESSION:ROLE-ANY`
   * Display name: `Login to Snowflake`
   * Description: `This scope allows the user to login to Snowflake as their current user`
   * Who can consent: Admins and users

---

## Step 3: Enable Device Code Flow for Public Clients

### Purpose:

Configure the app to support **device code flow**—an OAuth grant type that works without needing a browser redirect.

### How to:

1. Go to the **Authentication** tab
2. Click **“Add a platform”** → Choose **Mobile and desktop applications**
3. Enter the redirect URI:
   `msal<client-id>://auth`
   *(e.g., `msal8cb3c074-6a4b-4a07-9dff-3ae1b7ad0657://auth`)*
4. Under **Advanced settings**, enable:
    *Allow public client flows*

---

## Step 4: Create External OAuth Security Integration in Snowflake

### Purpose:

This configures Snowflake to **trust tokens** issued by Entra ID and map claims in the token to Snowflake users.

### How to:

```sql
CREATE OR REPLACE SECURITY INTEGRATION azure_snowflake_oauth
    TYPE = external_oauth
    ENABLED = TRUE
    EXTERNAL_OAUTH_TYPE = azure
    EXTERNAL_OAUTH_ISSUER = 'https://sts.windows.net/<tenant-id>/'
    EXTERNAL_OAUTH_AUDIENCE_LIST = ('api://<client-id>')
    EXTERNAL_OAUTH_JWS_KEYS_URL = 'https://login.microsoftonline.com/<tenant-id>/discovery/v2.0/keys'
    EXTERNAL_OAUTH_TOKEN_USER_MAPPING_CLAIM = 'unique_name'
    EXTERNAL_OAUTH_SNOWFLAKE_USER_MAPPING_ATTRIBUTE = 'EMAIL_ADDRESS'
    EXTERNAL_OAUTH_ANY_ROLE_MODE = 'ENABLE';
```

Replace `<tenant-id>` and `<client-id>` with your actual Azure values.

### Mapping Logic:

Snowflake will match the value of the `unique_name` claim in the Azure token to a Snowflake user whose `EMAIL_ADDRESS` matches it.

---

## Step 5: Install Required Python Libraries in Databricks

```python
%pip install snowflake-connector-python msal
```

---

## Step 6: Acquire OAuth Access Token Using MSAL

###  Purpose:

Use the MSAL library to request an access token using the **device code flow**.

### Code:

```python
import msal

client_id = "<client-id>"
authority = "https://login.microsoftonline.com/<tenant-id>"
scopes = ["api://<client-id>/SESSION:ROLE-ANY"]

app = msal.PublicClientApplication(client_id=client_id, authority=authority)

flow = app.initiate_device_flow(scopes=scopes)
if "user_code" not in flow:
    raise Exception("Device flow initiation failed")

print("Please go to", flow["verification_uri"], "and enter the code", flow["user_code"])

result = app.acquire_token_by_device_flow(flow)
if "access_token" not in result:
    raise Exception("Authentication failed: " + str(result.get("error_description")))

access_token = result["access_token"]
```

---

## Step 7 (Optional): Verify the Token

You can verify the token in Snowflake or manually via JWT inspection:

### Snowflake Token Check:

```sql
SELECT SYSTEM$VERIFY_EXTERNAL_OAUTH_TOKEN('<access_token>');
```

### Decode the Token:

Use [https://jwt.ms](https://jwt.ms) to verify:

* `aud` → matches `api://<client-id>`
* `iss` → matches your tenant’s issuer
* `unique_name` → matches Snowflake user’s `EMAIL_ADDRESS`

---

## Step 8: Connect to Snowflake Using OAuth Token

### Code:

```python
import snowflake.connector

conn = snowflake.connector.connect(
    account='<snowflake-account>',  # e.g., xyz12345.west-us-2.azure
    authenticator='oauth',
    token=access_token,
    warehouse='<warehouse>',
    database='<database>',
    schema='<schema>'
)

cursor = conn.cursor()
cursor.execute("SELECT current_user(), current_role(), current_database(), current_timestamp(), current_version()")
print(cursor.fetchall())
cursor.close()
conn.close()
```

Replace placeholder values like `<snowflake-account>`, `<warehouse>`, etc., with your actual config.

---

## Common Errors & Fixes

| Error                                                                                                   | Reason                                                 | Fix                                                                                 |
| ------------------------------------------------------------------------------------------------------- | ------------------------------------------------------ | ----------------------------------------------------------------------------------- |
| `Invalid OAuth Token`                                                                                   | Token has wrong audience or issuer                     | Verify audience (`aud`) and issuer (`iss`) match Snowflake integration              |
| `User does not exist`                                                                                   | `unique_name` in token doesn't map to Snowflake user   | Ensure the user’s email in Entra ID matches `EMAIL_ADDRESS` in Snowflake            |
| `Insufficient privileges`                                                                               | Token user doesn’t have role access                    | Confirm the user has the required role, and `EXTERNAL_OAUTH_ANY_ROLE_MODE = ENABLE` |
| ❗**Network Error**:<br>`Failed to connect to DB: ... Incoming request with IP/Token ... is not allowed` | Databricks IP is blocked by Snowflake's network policy | **Temporary Fix (for testing):** Allow all IPs for user                             |

```sql
CREATE NETWORK POLICY allow_all_ips ALLOWED_IP_LIST = ('0.0.0.0/0');
ALTER USER <username> SET NETWORK_POLICY = allow_all_ips;
-- Remove when done testing
ALTER USER <username> UNSET NETWORK_POLICY;
```
