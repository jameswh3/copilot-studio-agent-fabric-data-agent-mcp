# Copilot Studio Agent -- Fabric Data Agent MCP

This project documents and automates the On-Behalf-Of (OBO) setup for connecting Microsoft Fabric Data Agents as MCP endpoints to Copilot Studio agents.

> **Important:** The Copilot Studio MCP wizard defaults to standard OAuth 2.0 authentication. This approach requires users to frequently re-authenticate through the connection manager, creating a frustrating user experience. This OBO implementation uses managed identity and federated credentials instead, which enables seamless token exchange without exposing secrets. Once a user authenticates and creates the connection, it can be reused reliably without repeated login cycles.

## Files

Core setup:

- `setup_fabric_mcp_copilot_studio_obo.ipynb`

Repository guide:

- `README.md`

## Shared Prerequisites

**Required for both paths:**

- Rights to create Entra app registrations
- Rights to grant admin consent
- A published Microsoft Fabric Data Agent
- Permission to create or edit a Power Platform custom connector
- Permission to add tools in Copilot Studio if you plan to bind the connector there

**Required only for Path A (Automated):**

- Python 3.8 or later
- Azure CLI installed

## Shared .env Configuration

The `.env` file is only used in Path A (Automated). If using Path B (Manual), skip this section.

Example values:

```dotenv
TENANT_ID=<your-tenant-id>
SERVICE_APP_ID=<service-app-registration-client-id>
CONNECTOR_CLIENT_ID=<connector-app-registration-client-id>
AZURE_API_CONNECTIONS_SP_ID=fe053c5f-3692-4f14-aef2-ee34fc081cae
MCP_SERVER_URL=https://api.fabric.microsoft.com/v1/mcp/workspaces/<workspace-id>/dataagents/<dataagent-id>/agent
SWAGGER_OUTPUT_FILE=workingswagger-obo-template.yaml
```

Keep `.env` local and uncommitted.

## Quick Start: Choose Your Path

You have two options to set up OBO and the two Entra app registrations:

- **Path A (Automated)**: Run the notebook to automate Entra configuration.
- **Path B (Manual)**: Follow portal step-by-step instructions in the Azure portal.

Both paths converge at **Power Platform custom connector configuration**, which requires portal access and cannot be automated.

### Prerequisites (Both Paths)

- Tenant admin rights to create app registrations and grant admin consent.
- A published Microsoft Fabric Data Agent endpoint.
- Permission to create or edit a Power Platform custom connector.

### Path A: Automated Setup (Notebook)

Use this path if you have end-to-end tenant access and want to automate Entra app registration.

**1. Create app registrations manually (Portal)**

Before running the notebook, you must create two empty app registrations in Entra:

1. Create first app registration (Service app):
   - Single tenant
   - Save the Application ID as `SERVICE_APP_ID`

2. Create second app registration (Connector app):
   - Single tenant
   - Save the Application ID as `CONNECTOR_CLIENT_ID`

**2. Set up environment**

```powershell
python -m venv .venv
.venv\Scripts\activate
```

**3. Create .env file**

Create a `.env` file in the repository root with these values:

```dotenv
TENANT_ID=<your-tenant-id>
SERVICE_APP_ID=<service-app-client-id>
CONNECTOR_CLIENT_ID=<connector-app-client-id>
MCP_SERVER_URL=https://api.fabric.microsoft.com/v1/mcp/workspaces/<workspace-id>/dataagents/<dataagent-id>/agent
```

**4. Sign in with Azure CLI**

```powershell
az login --tenant <TENANT_ID> --use-device-code --scope https://graph.microsoft.com/.default
```

**5. Run notebook cells in order**

Open `setup_fabric_mcp_copilot_studio_obo.ipynb` and run these cells:

| Cell # | What it does |
|--------|---------------|
| 1 | Install dependencies: `requests`, `azure-identity`, `python-dotenv` |
| 2 | Load `.env`, validate values, set up Graph helpers |
| 3 | Configure service app (identifier URI, scope) and connector app (identifier URI, scope, permissions, preauth) |
| 4 | Verify Entra state; print next manual steps |
| 5 | Generate Swagger 2.0 file for Power Platform import |
| 6 | Print Power Platform connector security configuration checklist |
| 7 | Print test payloads for Power Platform Test tab |

After cells 1-4 complete successfully, proceed to **Common Steps: Power Platform Configuration** below.

### Path B: Manual Setup (Portal)

Use this path if you do not have end-to-end access, have security policies requiring manual approval, or prefer to do portal-based configuration.

No `.env` or notebook required for this path; all steps use the Azure portal.

#### B1 - Create the Service App Registration

1. In the Azure portal, go to **Entra ID > App registrations > New registration**.
2. Create a new app registration:
   - Name: `Fabric MCP Service App` (or similar)
   - Supported account types: Single tenant
3. After creation, open the app and note the **Application (client) ID** (this is your `SERVICE_APP_ID`).
4. Go to **Manage > Expose an API**:
   - Click **Set** next to Application ID URI
   - Accept the generated `api://{SERVICE_APP_ID}` URI
   - Click **Save**
5. Under Expose an API, click **Add a scope**:
   - Scope name: `access_as_user`
   - Admin consent display name: `Allow connector to call service on behalf of users`
   - Admin consent description: `Allows delegated access to the MCP service on behalf of users`
   - User consent display name: `Allow connector to act on your behalf`
   - User consent description: `Allows delegated access to the MCP service on your behalf`
   - State: Enabled
   - Click **Add scope**

#### B2 - Create the Connector App Registration

1. In the Azure portal, go to **Entra ID > App registrations > New registration**.
2. Create a new app registration:
   - Name: `Fabric MCP Connector App` (or similar)
   - Supported account types: Single tenant
3. After creation, note the **Application (client) ID** (this is your `CONNECTOR_CLIENT_ID`).
4. Go to **Manage > Expose an API**:
   - Click **Set** next to Application ID URI
   - Accept the generated `api://{CONNECTOR_CLIENT_ID}` URI
   - Click **Save**
5. Under Expose an API, click **Add a scope**:
   - Scope name: `access_as_user`
   - Admin consent display name: `Allow Azure API Connections to obtain tokens on behalf of users`
   - Admin consent description: `Allows Azure API Connections to obtain tokens on behalf of the user`
   - User consent display name: `Allow connector to act on your behalf`
   - User consent description: `Allows Azure API Connections to access resources on your behalf`
   - State: Enabled
   - Click **Add scope**

#### B3 - Add Permissions to Connector App

1. Open the Connector app registration.
2. Go to **Manage > API permissions**.
3. Click **Add a permission**:
   - Select **APIs my organization uses** tab
   - Search for and select your Service app (by name or `SERVICE_APP_ID`)
   - Select **Delegated permissions**
   - Check `access_as_user`
   - Click **Add permissions**
4. Click **Add a permission** again:
   - Select **APIs my organization uses** tab
   - Search for `Microsoft Fabric`
   - Select **Delegated permissions**
   - Check `DataAgent.Execute.All` and `Item.Read.All`
   - Click **Add permissions**
5. Click **Grant admin consent for [tenant name]** and confirm.

#### B4 - Preauthorize Azure API Connections

1. Open the Connector app registration.
2. Go to **Manage > Expose an API**.
3. Under Authorized client applications, click **Add a client application**:
   - Client ID: `fe053c5f-3692-4f14-aef2-ee34fc081cae` (Azure API Connections)
   - Authorized scopes: Check `api://{CONNECTOR_CLIENT_ID}/access_as_user`
   - Click **Add application**

#### B5 - Verify Entra Configuration

Verify these settings are present before moving to Power Platform configuration:

- Service app has `api://{SERVICE_APP_ID}` identifier URI
- Service app has `access_as_user` scope
- Connector app has `api://{CONNECTOR_CLIENT_ID}` identifier URI
- Connector app has `access_as_user` scope
- Connector app has delegated permission to service app `access_as_user` scope
- Connector app has delegated permissions to Fabric (`DataAgent.Execute.All`, `Item.Read.All`)
- Azure API Connections (`fe053c5f-...`) is preauthorized on connector app with `api://{CONNECTOR_CLIENT_ID}/access_as_user` scope

## What The OBO Flow Creates

The OBO notebook uses a two-app model:

- Service app registration: the resource app that exposes `api://<SERVICE_APP_ID>/access_as_user`
- Connector app registration: the client app used by the Power Platform custom connector

The connector app is configured to request delegated access to:

- the service app scope `access_as_user`
- Microsoft Fabric delegated scopes needed by the Data Agent endpoint

The notebook also preauthorizes Azure API Connections on the connector app so Power Platform can participate in the token flow.


## Common Steps: Power Platform Configuration

These steps apply regardless of which path (A or B) you followed for Entra app registration.

### Generate the Swagger 2.0 File

**If you used Path A (Notebook):**

Run cell 5 in the notebook to automatically generate `workingswagger-obo-template.yaml`. Skip to the next subsection.

**If you used Path B (Manual):**

Create a Swagger 2.0 file manually. Use this template and replace placeholders:

```yaml
swagger: '2.0'
info:
  title: Fabric MCP Server
  description: Swagger template for a Copilot Studio custom connector that calls a Fabric Data Agent MCP endpoint.
  version: 1.0.0
host: api.fabric.microsoft.com
basePath: /
schemes:
  - https
paths:
  /v1/mcp/workspaces/<WORKSPACE_ID>/dataagents/<DATAAGENT_ID>/agent:
    post:
      responses:
        '200':
          description: Immediate Response
      x-ms-agentic-protocol: mcp-streamable-1.0
      operationId: InvokeServer
      summary: Fabric MCP Server
      description: Invokes the MCP server endpoint for the configured Fabric Data Agent.
securityDefinitions:
  oauth2-auth:
    type: oauth2
    flow: accessCode
    authorizationUrl: https://login.microsoftonline.com/<TENANT_ID>/oauth2/v2.0/authorize
    tokenUrl: https://login.microsoftonline.com/<TENANT_ID>/oauth2/v2.0/token
    scopes:
      https://api.fabric.microsoft.com/.default: https://api.fabric.microsoft.com/.default
security:
  - oauth2-auth:
      - https://api.fabric.microsoft.com/.default
```

Replace:
- `<WORKSPACE_ID>`: Your Fabric workspace ID
- `<DATAAGENT_ID>`: Your Fabric Data Agent ID
- `<TENANT_ID>`: Your Microsoft Entra tenant ID

### Import the Custom Connector

1. In Power Automate or Power Platform custom connectors, create a new custom connector from Swagger.
2. Import the Swagger 2.0 file you generated (or created manually).
3. Save the connector.

At this point, the connector exists but still needs security configuration.

### Configure Connector Security (OBO)

1. Open the connector and go to the **Security** tab.
2. Configure these values:
   - Authentication type: **OAuth 2.0**
   - Identity Provider: **Azure Active Directory**
   - Secret option: **Use managed identity**
   - Enable on-behalf-of login: **true**
   - Client ID: `<CONNECTOR_CLIENT_ID>`
   - Tenant ID: `<TENANT_ID>` (not `common`)
   - Resource URL: `https://api.fabric.microsoft.com`
   - Scope: `https://api.fabric.microsoft.com/.default`
3. Go to the **Definition** tab and confirm:
   - Operation: **InvokeServer**
   - Verb: **POST**
   - URL: Your MCP server URL (e.g., `https://api.fabric.microsoft.com/v1/mcp/workspaces/<WORKSPACE_ID>/dataagents/<DATAAGENT_ID>/agent`)
   - `x-ms-agentic-protocol: mcp-streamable-1.0`
4. Save the connector once.

### Add Redirect URI and Federated Credential

After saving, copy these values from the connector details:

- Redirect URL
- Managed identity Issuer
- Managed identity Subject

Then add them to your Connector app registration in Entra:

1. Open the Connector app registration.
2. Go to **Manage > Authentication**:
   - Under Redirect URIs, click **Add URI**
   - Paste the Redirect URL from connector details
   - Click **Save**
3. Go to **Manage > Certificates and secrets**:
   - Click the **Federated credentials** tab
   - Click **Add credential**
   - Configure:
     - Scenario: **Other issuer**
     - Issuer: paste the Managed identity Issuer value
     - Subject: paste the Managed identity Subject value
     - Audience: `api://AzureADTokenExchange`
   - Click **Add**

**Service app registration (resource app):** No action required in this step.

### Recreate Connection

After configuring federated credentials and security settings:

1. Delete any existing connections for this connector.
2. Create a new connection for the connector.
3. Complete sign-in when prompted.

This avoids stale token audience issues after security changes.

### Validate Connection and Test

1. Open the connector Test tab and select the `InvokeServer` operation.
2. Turn on **Raw Body**.
3. Send the initialize request:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "initialize",
  "params": {
    "protocolVersion": "2024-11-05",
    "capabilities": {},
    "clientInfo": {
      "name": "power-platform-test",
      "version": "1.0.0"
    }
  }
}
```

Expected result: HTTP 200 with `jsonrpc: 2.0`, `result.protocolVersion`, and `result.serverInfo`.

4. Send the tools/list request:

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "tools/list",
  "params": {}
}
```

Expected result: HTTP 200 with the list of tools from the Fabric Data Agent.

**Important notes:**

- `InvokeServer` is the Power Platform operation name.
- The JSON body `method` must be an MCP method such as `initialize`, `tools/list`, or `tools/call`.
- `tools/call` requires a `user_question` parameter; if the Fabric Data Agent requires it, you can test with `initialize` and `tools/list` to validate the connection is working.

### Add Connector to Copilot Studio

After the connector validates successfully with `initialize` and `tools/list`:

1. Open your Copilot Studio agent.
2. Add the custom connector as a tool.
3. Select the MCP operation.
4. Publish and test the agent.

## Troubleshooting

- Token acquisition fails: verify Azure API Connections is preauthorized on the connector app.
- Redirect URL invalid: ensure the exact connector Redirect URL is present on the connector app.
- Unauthorized during `initialize`: verify the connector Security tab uses `https://api.fabric.microsoft.com` as Resource URL and `https://api.fabric.microsoft.com/.default` as Scope for the direct Fabric test path.
- Test still uses old token audience: delete and recreate the connector connection after Security changes.
- `invokeServer` was sent as the MCP method: use `initialize`, `tools/list`, or `tools/call` in the JSON body.
- Test tab request shape is wrong: enable Raw Body and send the full JSON-RPC payload.

## References

- [Microsoft: Configure OBO authentication for custom connectors](https://learn.microsoft.com/en-us/microsoft-copilot-studio/advanced-custom-connector-on-behalf-of)
- [Microsoft CAT Blog: You Probably Don't Need Manual Auth](https://microsoft.github.io/mcscatblog/posts/you-dont-need-manual-auth/)
- [Microsoft CAT Blog: Seamless SSO with Custom Connectors](https://microsoft.github.io/mcscatblog/posts/obo-for-custom-connectors/)
- [OAuth 2.0 On-Behalf-Of Flow](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-on-behalf-of-flow)
