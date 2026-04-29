# Fabric Data Agent MCP to Copilot Studio Setup - OBO Pattern (Browser-Only Guide)

**Authentication Pattern: On-Behalf-Of (OBO) with two app registrations (Connector App + Service App) and Azure API Connections managed identity**

This guide is aligned to the pattern implemented in your environment and the Microsoft OBO guidance for custom APIs/MCP servers.

Working pattern in this guide:

- Service app registration (resource/server app): defines the OBO permission boundary in Entra; exposes `access_as_user`
- Connector app registration (client app): requests delegated access to the service app scope and Fabric scopes
- Connector app Expose an API + Azure API Connections authorized client app
- Custom connector with managed identity and OBO enabled; runtime Resource URL = `https://api.fabric.microsoft.com`
- Federated credential on the connector app using the custom connector managed identity values
- Swagger 2.0 MCP definition

**Runtime note:** The service app and connector app define the Entra permission model. At runtime, the connector calls the Fabric MCP endpoint directly. The token audience is `https://api.fabric.microsoft.com`, not the service app. If you later place your own service in front of Fabric, the service app becomes the runtime resource.

**Spec Format**: Custom connector specs for this flow must use **Swagger 2.0** format, which is required for Power Platform and Copilot Studio compatibility.

This guide is fully standalone and does not require running any notebook.

## Prerequisites

Before starting:

- Your Fabric Data Agents are published.
- You have rights to create app registrations in Entra ID.
- You have rights to grant admin consent.
- You can edit tools in Copilot Studio and Power Automate custom connectors.

## Architecture Overview

OBO flow with direct Fabric runtime audience:

```
User (in M365 Copilot)
    |
    v
Copilot Studio (via Custom Connector)
    |
   | [user identity]
    v
Azure API Connections (managed identity + token exchange)
    |
   | [token for https://api.fabric.microsoft.com]
    v
Fabric Data Agent MCP Endpoint
    v
Data returned to user
```

Note: The service app and connector app both exist in Entra and define the OBO
permission model. The connector calls Fabric directly at runtime.

---

## Step 1 - Create the Service App Registration (Resource/Server App)

This app represents the resource scope used for OBO.

**In Entra admin center:**

1. Go to **App registrations** -> **New registration**.
2. Name it (for example): `Fabric Data Agent Service`.
3. Supported account types: **Accounts in this organizational directory only**.
4. Select **Register**.
5. Save:
   - Application (client) ID (this is your `SERVICE_APP_ID`)
   - Directory (tenant) ID

### Step 1a - Expose a scope on the service app

**In service app > Expose an API:**

1. Select **Add a scope**.
2. Accept the default Application ID URI (for example `api://<service-app-id>`).
3. Configure scope:
   - Scope name: `access_as_user`
   - Who can consent: Admins and users
   - Admin consent display name: `Allow connector to call service on behalf of users`
   - Admin consent description: `Allows delegated access to the MCP service on behalf of users`
   - State: **Enabled**
4. Select **Add scope**.
5. Save the full scope value:
   - `api://<service-app-id>/access_as_user`

---

## Step 2 - Create the Connector App Registration (Client App)

This app represents the custom connector client.

**In Entra admin center:**

1. Go to **App registrations** -> **New registration**.
2. Name it (for example): `Copilot Studio - Fabric MCP Connector OBO`.
3. Supported account types: **Accounts in this organizational directory only**.
4. Select **Register**.
5. Save:
   - Application (client) ID (this is your `CONNECTOR_CLIENT_ID`)

### Step 2a - Add delegated permission to the service app scope

**In connector app > API permissions:**

1. Select **Add a permission**.
2. Select **APIs my organization uses**.
3. Search for your **service app** (from Step 1) and select it.
4. Select **Delegated permissions**.
5. Select `access_as_user`.
6. Select **Add permissions**.
7. Select **Grant admin consent for [your org]**.

### Step 2b - Expose API on the connector app (required for Azure API Connections)

**In connector app > Expose an API:**

1. Select **Add a scope**.
2. Accept default Application ID URI (for example `api://<connector-app-id>`).
3. Configure scope:
   - Scope name: `access_as_user`
   - Admin consent display name: `Allow Azure API Connections to obtain tokens on behalf of users`
   - Admin consent description: `Allows Azure API Connections to obtain tokens on behalf of the user`
   - User consent display name: `Allow connector to act on your behalf`
   - User consent description: `Allows Azure API Connections to access resources on your behalf`
   - State: **Enabled**
4. Select **Add scope**.

### Step 2c - Authorize Azure API Connections client app

**In connector app > Expose an API > Authorized client applications:**

1. Select **Add a client application**.
2. Enter client ID: `fe053c5f-3692-4f14-aef2-ee34fc081cae`.
3. Select scope `api://<connector-app-id>/access_as_user`.
4. Select **Add application**.

---

## Step 3 - Prepare Fabric Data Agents

In Fabric:

1. Open each Data Agent you want to use.
2. Confirm it is **published**.
3. Open Settings -> **Model Context Protocol**.
4. Copy the MCP endpoint URL for each agent.

Keep a list: Agent Name, Workspace, MCP URL.

---

## Step 4 - Use the Swagger 2.0 Template

Use the template below and replace tenant, workspace, data agent, and service app ID values.

```yaml
swagger: '2.0'
info:
  title: <Your MCP Server Name>
  description: >-
    <Describe what this MCP tool does for users.>
  version: 1.0.0
host: api.fabric.microsoft.com
basePath: /
schemes:
  - https
paths:
  /v1/mcp/workspaces/<workspace-id>/dataagents/<data-agent-id>/agent:
    post:
      responses:
        '200':
          description: Immediate Response
      x-ms-agentic-protocol: mcp-streamable-1.0
      operationId: InvokeServer
      summary: <Your MCP Server Name>
      description: >-
        <Describe what this MCP tool does for users.>
securityDefinitions:
  oauth2-auth:
    type: oauth2
    flow: accessCode
    authorizationUrl: https://login.microsoftonline.com/<tenant-id>/oauth2/v2.0/authorize
    tokenUrl: https://login.microsoftonline.com/<tenant-id>/oauth2/v2.0/token
    scopes:
      https://api.fabric.microsoft.com/.default: https://api.fabric.microsoft.com/.default
security:
  - oauth2-auth:
      - https://api.fabric.microsoft.com/.default
```

Keep this structure exactly:

- `basePath: /`
- Full MCP endpoint path under `paths`
- `x-ms-agentic-protocol: mcp-streamable-1.0`
- No manual JSON-RPC body parameter definitions

---

## Step 5 - Create the Custom Connector

**In Power Automate > Custom connectors:**

1. Create a new custom connector from Swagger 2.0.
2. Import the filled template from Step 4.
3. Save the connector.

---

## Step 6 - Configure OBO Security on the Custom Connector

**In custom connector > Security:**

1. Authentication type: **OAuth 2.0**
2. Identity Provider: **Azure Active Directory**
3. Secret option: **Use managed identity**
4. Configure values:

| Field | Value |
|---|---|
| Client ID | `CONNECTOR_CLIENT_ID` from Step 2 |
| Authorization URL | `https://login.microsoftonline.com` |
| Tenant ID | Your tenant ID |
| Resource URL | `https://api.fabric.microsoft.com` |
| Enable on-behalf-of login | `true` |
| Scope | `https://api.fabric.microsoft.com/.default` |

5. Select **Update connector**.
6. Copy the generated **Redirect URL** and **Managed identity** values (Issuer and Subject) from the Details page.

### Step 6a - Add redirect URL to connector app registration

**In Entra > connector app > Authentication:**

1. Select **Add a platform** -> **Web**.
2. Paste the Redirect URL copied from the custom connector.
3. Save.

---

## Step 7 - Add Federated Credential on Connector App

**In Entra > connector app > Certificates & secrets > Federated credentials:**

1. Add a new federated credential.
2. Use:
   - Issuer: value from connector Details page
   - Subject: value from connector Details page
   - Audience: `api://AzureADTokenExchange`
3. Save.

If connector managed identity values change (for example after recreating connector), update or add federated credential entries accordingly.

---

## Step 8 - Create a Fresh Connector Connection

After Security/federated credential changes:

1. Delete older connections for this connector.
2. Create one new connection.
3. Sign in and complete consent.

---

## Step 9 - Add Connector as MCP Tool in Copilot Studio

**In Copilot Studio > Agent > Tools:**

1. Select **Add a tool**.
2. Select your custom connector.
3. Select MCP action (for example `InvokeServer`).
4. If search does not find it, scroll the full list.

If connector definition changes later, recreate connection and rebind the tool.

---

## Step 10 - Create Solution for Export/Import

1. Open your agent in Copilot Studio.
2. Create or open a custom solution.
3. Add the agent and connector dependencies.
4. Use this solution for ALM/export-import.

---

## Step 11 - Finalize Agent Configuration

1. Add instructions for when to use each MCP tool.
2. Add other knowledge sources if needed.
3. Publish the agent.
4. Test in target channels.

---

## Verification Checklist

- [ ] Service app registration created
- [ ] Service app scope `access_as_user` exposed
- [ ] Connector app registration created
- [ ] Connector app has delegated permission to service app `access_as_user`
- [ ] Admin consent granted
- [ ] Connector app exposes its own `access_as_user` scope
- [ ] Azure API Connections client app authorized on connector app
- [ ] Swagger 2.0 template uses Fabric runtime scope (`https://api.fabric.microsoft.com/.default`)
- [ ] Custom connector security uses Resource URL `https://api.fabric.microsoft.com`
- [ ] OBO enabled in connector security
- [ ] Redirect URL added in connector app Authentication
- [ ] Federated credential on connector app matches managed identity issuer/subject
- [ ] Fresh connector connection created
- [ ] MCP tool bound in Copilot Studio

---

## Troubleshooting

**OBO consent or token exchange fails:**

- Verify connector app delegated permission to service app scope is granted and consented.
- Verify connector app has Azure API Connections authorized client application.
- Verify federated credential Issuer and Subject exactly match connector managed identity.

**Redirect URL errors:**

- Ensure redirect URL in connector app Authentication exactly matches custom connector Details.
- Remove stale/old redirect URLs and retry.

**401/Unauthorized from MCP endpoint:**

- Verify connector security Resource URL is `https://api.fabric.microsoft.com`.
- Verify Scope is `https://api.fabric.microsoft.com/.default`.
- Verify OBO is enabled.
- Verify user has access to the Fabric Data Agent and agent is published.

---

## Using the Same App Registrations for Multiple Connectors

Yes. You can reuse the same service app and connector app across multiple custom connectors.

Important details:

- Shared across connectors:
  - Service app registration
  - Connector app registration
  - Connector client ID
  - Service app scope value
- Unique per connector:
  - Custom connector definition
  - Connection object
  - Managed identity Issuer/Subject values
  - Federated credential entry on connector app

For each additional connector:

1. Create/import a new Swagger definition.
2. Configure Security with same connector client ID, Resource URL `https://api.fabric.microsoft.com`, and Scope `https://api.fabric.microsoft.com/.default`.
3. Copy that connector's managed identity values.
4. Add another federated credential entry on the connector app for that identity.
5. Create a fresh connection and bind in Copilot Studio.

---

## References

- [Microsoft: Configure OBO authentication for custom connectors](https://learn.microsoft.com/en-us/microsoft-copilot-studio/advanced-custom-connector-on-behalf-of)
- [OAuth 2.0 On-Behalf-Of Flow](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-on-behalf-of-flow)
