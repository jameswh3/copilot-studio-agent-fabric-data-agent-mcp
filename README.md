# Copilot Studio Agent -- Fabric Data Agent MCP

This project provides two authentication patterns for connecting Microsoft Fabric Data Agents as MCP endpoints to Copilot Studio agents:

1. OAuth 2.0 (Manual): single app registration, manual secret handling, simpler setup.
2. On-Behalf-Of (OBO): two app registrations, delegated permission model, better alignment with enterprise and ALM scenarios.

Use the manual path when you want the smallest setup surface. Use the OBO path when you want a structured Entra permission model and Power Platform-friendly connector auth.

## Files

Manual OAuth:

- `setup_fabric_mcp_copilot_studio_manual_auth.ipynb`
- `MCP_COPILOT_STUDIO_SETUP_MANUAL_AUTH_PLAIN_LANGUAGE.md`

OBO:

- `setup_fabric_mcp_copilot_studio_obo.ipynb`
- `MCP_COPILOT_STUDIO_SETUP_OBO_PLAIN_LANGUAGE.md`

Reference:

- `OAUTH_SETUP_COMPARISON.md`

## Swagger 2.0 Compatibility

Power Platform and Copilot Studio custom connectors require Swagger 2.0, not OpenAPI 3.x.

The notebook-generated specs use the tested MCP connector shape:

- `swagger: '2.0'`
- `host: api.fabric.microsoft.com`
- `basePath: /`
- full MCP endpoint path under `paths`
- `x-ms-agentic-protocol: mcp-streamable-1.0` on the `post` operation

## Shared Prerequisites

- Python 3.8 or later
- Azure CLI installed
- rights to create Entra app registrations
- rights to grant admin consent
- a published Microsoft Fabric Data Agent
- permission to create or edit a Power Platform custom connector
- permission to add tools in Copilot Studio if you plan to bind the connector there

## Shared .env Configuration

The notebooks use a local `.env` file for persistent configuration.

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

## Quick Start

Manual OAuth:

1. Create and activate a virtual environment.
2. Open `setup_fabric_mcp_copilot_studio_manual_auth.ipynb`.
3. Run the notebook.

OBO:

1. Create and activate a virtual environment.
2. Open `setup_fabric_mcp_copilot_studio_obo.ipynb`.
3. Run the notebook.

Basic setup commands:

```powershell
python -m venv .venv
.venv\Scripts\activate
```

## OBO Notebook Walkthrough

This section is the standalone walkthrough for the flow implemented in `setup_fabric_mcp_copilot_studio_obo.ipynb`.

It follows the same sequence as the notebook:

1. Install notebook dependencies.
2. Load local configuration from `.env`.
3. Configure the two Entra app registrations used for the OBO pattern.
4. Verify the live Entra state.
5. Generate a Swagger 2.0 custom connector definition.
6. Configure the Power Platform custom connector.
7. Copy redirect URI and managed identity values back into Entra.
8. Test the connector with MCP JSON-RPC requests.

This walkthrough can be used without running the notebook, but it stays aligned to the notebook's actual behavior and outputs.

## What The OBO Flow Creates

The OBO notebook uses a two-app model:

- Service app registration: the resource app that exposes `api://<SERVICE_APP_ID>/access_as_user`
- Connector app registration: the client app used by the Power Platform custom connector

The connector app is configured to request delegated access to:

- the service app scope `access_as_user`
- Microsoft Fabric delegated scopes needed by the Data Agent endpoint

The notebook also preauthorizes Azure API Connections on the connector app so Power Platform can participate in the token flow.

## Important Runtime Note

There are two related ideas in this setup:

- The Entra OBO app registration model uses both the service app and connector app.
- The connector test flow that worked against the live Fabric endpoint used Fabric as the runtime audience.

That means the service app is part of the Entra permission model created by the notebook, but the final direct connector test talks to Fabric and uses:

- Resource URL: `https://api.fabric.microsoft.com`
- Scope: `https://api.fabric.microsoft.com/.default`

If you later place your own service endpoint in front of Fabric, that service app becomes the runtime resource as well.

## Step 1 - Create the Service App Registration

Create a new Entra app registration for the service app.

Use these settings:

- Supported account types: Single tenant
- Save the Application (client) ID as `SERVICE_APP_ID`

Then go to Expose an API and configure:

- Application ID URI: `api://<SERVICE_APP_ID>`
- Scope name: `access_as_user`
- Admin consent display name: `Allow connector to call service on behalf of users`
- Admin consent description: `Allows delegated access to the MCP service on behalf of users`
- User consent display name: `Allow connector to act on your behalf`
- User consent description: `Allows delegated access to the MCP service on your behalf`

This is the resource boundary for the two-app OBO model.

## Step 2 - Create the Connector App Registration

Create a second Entra app registration for the connector app.

Use these settings:

- Supported account types: Single tenant
- Save the Application (client) ID as `CONNECTOR_CLIENT_ID`

Then configure two things on the connector app.

First, under API permissions, add delegated permission to the service app:

- API: your service app
- Delegated permission: `access_as_user`

Second, under Expose an API, configure:

- Application ID URI: `api://<CONNECTOR_CLIENT_ID>`
- Scope name: `access_as_user`
- Admin consent display name: `Allow Azure API Connections to obtain tokens on behalf of users`
- Admin consent description: `Allows Azure API Connections to obtain tokens on behalf of the user`
- User consent display name: `Allow connector to act on your behalf`
- User consent description: `Allows Azure API Connections to access resources on your behalf`

Then preauthorize Azure API Connections:

- Authorized client application: `fe053c5f-3692-4f14-aef2-ee34fc081cae`
- Authorized scope: `api://<CONNECTOR_CLIENT_ID>/access_as_user`

Grant admin consent after adding permissions.

## Step 3 - Sign In With Azure CLI

Sign in before running the notebook or before reproducing its Graph operations:

```powershell
az login --tenant <TENANT_ID> --use-device-code --scope https://graph.microsoft.com/.default
```

The notebook uses Azure CLI authentication to call Microsoft Graph.

## Step 4 - Run The Notebook Setup Logic

Open `setup_fabric_mcp_copilot_studio_obo.ipynb` and run the first five cells in order.

What they do:

1. Install `requests`, `azure-identity`, and `python-dotenv`.
2. Load `.env`, validate required values, and create Graph helpers.
3. Configure the service app and connector app.
4. Resolve live permission IDs from Entra and Fabric.
5. Verify that the final state is correct.

The notebook applies these live changes:

- service app identifier URI becomes `api://<SERVICE_APP_ID>`
- service app exposes `access_as_user`
- connector app identifier URI becomes `api://<CONNECTOR_CLIENT_ID>`
- connector app requests delegated access to the service app scope
- connector app requests Fabric delegated scopes needed by the Data Agent endpoint
- connector app exposes its own `access_as_user` scope
- connector app preauthorizes Azure API Connections
- admin consent is requested for the connector app

## Step 5 - Verify The App Registration State

The notebook verify step checks:

- service app identifier URI exists
- service app scope `access_as_user` exists
- connector app identifier URI exists
- connector app scope `access_as_user` exists
- Azure API Connections is preauthorized on the connector app
- connector app has delegated permission to the service app
- connector app has the required Fabric delegated permissions
- federated credentials count on the connector app

Do not continue until those checks are successful.

## Step 6 - Generate The Swagger 2.0 File

Run the Swagger generation cell in the notebook.

It produces a Swagger 2.0 file using:

- your tenant ID for `authorizationUrl` and `tokenUrl`
- your `MCP_SERVER_URL` for the host and path
- the Fabric runtime scope (`https://api.fabric.microsoft.com/.default`) for the security definition

The generated file keeps the Power Platform-compatible MCP shape:

- `swagger: '2.0'`
- `host: api.fabric.microsoft.com`
- `basePath: /`
- full MCP endpoint path under `paths`
- `x-ms-agentic-protocol: mcp-streamable-1.0`
- operation name `InvokeServer`

## Step 7 - Import The Custom Connector

In Power Automate or Power Platform custom connectors:

1. Create a new custom connector from Swagger.
2. Import the generated Swagger 2.0 file.
3. Save the connector.

At this point, the connector exists but still needs final security wiring.

## Step 8 - Configure Connector Security

For the direct Fabric test path that worked, configure the Security tab with these values:

- Authentication type: OAuth 2.0
- Identity Provider: Azure Active Directory
- Secret option: Use managed identity
- Enable on-behalf-of login: true
- Client ID: `CONNECTOR_CLIENT_ID`
- Tenant ID: your tenant ID, not `common`
- Resource URL: `https://api.fabric.microsoft.com`
- Scope: `https://api.fabric.microsoft.com/.default`

On the Definition tab, confirm:

- Operation: `InvokeServer`
- Verb: `POST`
- URL: your `MCP_SERVER_URL`
- `x-ms-agentic-protocol: mcp-streamable-1.0`

Save the connector.

## Step 9 - Copy Values Back Into Entra

After the first save, copy these values from the connector details:

- Redirect URL
- Managed identity Issuer
- Managed identity Subject

Enter those values into the connector app registration in Entra.

Authentication page:

- Add the Redirect URL under Redirect URIs

Certificates and secrets > Federated credentials:

- Scenario: `Other issuer`
- Issuer: connector Managed identity Issuer
- Subject: connector Managed identity Subject
- Audience: `api://AzureADTokenExchange`

No update is required on the service app for this step.

## Step 10 - Recreate The Connector Connection

After changing Security or federated credential settings:

1. Delete older connections for the connector.
2. Create a fresh connection.
3. Complete sign-in again.

This avoids stale token audience issues.

## Step 11 - Test The Connector In Power Platform

Open the Test tab and choose the `InvokeServer` operation.

Turn on Raw Body.

Use these requests in order.

### Request 1 - initialize

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

Expected result:

- HTTP 200
- `jsonrpc: 2.0`
- `result.protocolVersion`
- `result.serverInfo`

### Request 2 - tools/list

```json
{
   "jsonrpc": "2.0",
   "id": 2,
   "method": "tools/list",
   "params": {}
}
```

Expected result:

- HTTP 200
- a tool list returned by the Fabric Data Agent

### Request 3 - tools/call

Replace the tool name with one returned by `tools/list` and provide the arguments that tool expects.

```json
{
   "jsonrpc": "2.0",
   "id": 3,
   "method": "tools/call",
   "params": {
      "name": "<tool-name-from-tools-list>",
      "arguments": {}
   }
}
```

Important:

- `InvokeServer` is the Power Platform operation name.
- The JSON body `method` must be an MCP method such as `initialize`, `tools/list`, or `tools/call`.

## Step 12 - Add The Connector To Copilot Studio

After the connector tests successfully:

1. Open your Copilot Studio agent.
2. Add the custom connector as a tool.
3. Choose the MCP operation.
4. Publish and test the agent.

## Troubleshooting

Manual OAuth issues:

- Connection fails at sign-in: check the redirect URI in the Entra app registration.
- Copilot Studio create flow throws a truncation-style error: shorten server name, description, and initial scope set.
- Permission denied: verify admin consent was granted on Fabric API scopes.

OBO issues:

- Token acquisition fails: verify Azure API Connections is preauthorized on the connector app.
- Redirect URL invalid: ensure the exact connector Redirect URL is present on the connector app.
- Unauthorized during `initialize`: verify the connector Security tab uses `https://api.fabric.microsoft.com` as Resource URL and `https://api.fabric.microsoft.com/.default` as Scope for the direct Fabric test path.
- Test still uses old token audience: delete and recreate the connector connection after Security changes.
- `invokeServer` was sent as the MCP method: use `initialize`, `tools/list`, or `tools/call` in the JSON body.
- Test tab request shape is wrong: enable Raw Body and send the full JSON-RPC payload.

## Comparison Matrix

| Aspect | Manual OAuth | OBO |
| --- | --- | --- |
| Complexity | Simple | Medium |
| App registrations | 1 | 2 |
| Token management | Manual or exposed | Framework-managed |
| Multi-resource support | Single, primarily Fabric | Yes |
| Seamless consent | No | Yes |
| ALM support | No | Yes |
| Enterprise alignment | Lower | Higher |

## References

- [Microsoft: Configure OBO authentication for custom connectors](https://learn.microsoft.com/en-us/microsoft-copilot-studio/advanced-custom-connector-on-behalf-of)
- [Microsoft CAT Blog: You Probably Don't Need Manual Auth](https://microsoft.github.io/mcscatblog/posts/you-dont-need-manual-auth/)
- [Microsoft CAT Blog: Seamless SSO with Custom Connectors](https://microsoft.github.io/mcscatblog/posts/obo-for-custom-connectors/)
- [OAuth 2.0 On-Behalf-Of Flow](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-on-behalf-of-flow)
