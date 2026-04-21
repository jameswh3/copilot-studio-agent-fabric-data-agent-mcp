# Fabric Data Agent MCP to Copilot Studio Setup (Browser-Only Guide)

This runbook describes what we set up using browser portals only:

- Fabric Data Agents as MCP endpoints
- Entra app registration for OAuth 2.0
- Copilot Studio MCP tool connections (client secret pasted directly)

Use this if your admins and makers are configuring everything in the web UI.

## Prerequisites

Before starting:

- Your Fabric Data Agents are published.
- You have rights to create or update an app registration in Entra.
- You have rights to grant admin consent (or know who does).
- You can edit tools in a Copilot Studio agent.

## 1) Prepare Fabric Data Agents

In Fabric:

1. Open each Data Agent you want to use.
2. Confirm it is published.
3. Open Settings > Model Context Protocol.
4. Copy and save the MCP endpoint URL for each agent.

Tip: Keep a short list with Agent Name, Workspace, and MCP URL.

## 2) Create or open the Copilot Studio solution

In Copilot Studio:

1. Open your agent.
2. Open the solution explorer for that agent, or create a custom solution if you are still using only the default solution.
3. Confirm this is the solution that will carry the agent for export and import between environments.

## 3) Create the Entra app registration

In Entra admin center:

1. Go to App registrations.
2. Create a new app registration.
3. Name it something like: Copilot Studio - Fabric MCP Connector.
4. Set supported account type to single tenant (same org).
5. Save the app.

Capture these values:

- Application (client) ID
- Directory (tenant) ID

## 4) Add API permissions

Still in the app registration:

1. Go to API permissions.
2. Add Microsoft Fabric delegated permissions:
3. Add DataAgent.Execute.All.
4. Add Item.Read.All.
5. Save changes.

## 5) Create client secret

In app registration > Certificates & secrets:

1. Create a new client secret.
2. Choose an expiration policy your organization accepts.
3. Copy the secret value immediately and store it securely.

Important: You can only copy the secret value once. You will paste it directly into Copilot Studio in step 7.

## 6) Grant admin consent

In app registration > API permissions:

1. Select Grant admin consent for your organization.
2. Confirm status shows granted for the required permissions.

## 7) Add MCP tools in Copilot Studio

In Copilot Studio:

1. Open your agent.
2. Go to Tools > Add a tool > New tool > Model Context Protocol.
3. Create one MCP tool per Fabric Data Agent.
4. For each tool, enter:
   - Server name
   - Server description
   - Server URL (the Fabric MCP endpoint)
5. Set authentication to OAuth 2.0 (Manual).
6. When prompted for the client secret, paste the value you copied in step 5.

## 8) Configure OAuth fields in Copilot Studio

Use values from your Entra app:

- Client ID: Application (client) ID
- Client Secret: the value you copied in step 5
- Authorization URL: https://login.microsoftonline.com/<tenant-id>/oauth2/v2.0/authorize
- Token URL: https://login.microsoftonline.com/<tenant-id>/oauth2/v2.0/token
- Refresh URL: same as token URL
- Scopes: https://api.fabric.microsoft.com/DataAgent.Execute.All https://api.fabric.microsoft.com/Item.Read.All openid profile

If the wizard throws a generic save error such as String or binary data would be truncated, try these safer values first:

- Keep Server name short and use only letters, digits, hyphens, and underscores.
- Keep Server description short.
- Make sure Authorization URL, Token URL, and Refresh URL all use the same tenant ID.
- Start with the minimum scope: https://api.fabric.microsoft.com/DataAgent.Execute.All

Use the same OAuth values for each MCP tool connection.

## 9) Register Redirect URI back in Entra

When you create each MCP connection in Copilot Studio, a Redirect URI is shown.

For each Redirect URI:

1. Copy it from Copilot Studio.
2. In Entra app registration, go to Authentication.
3. Add the URI under Web redirect URIs.
4. Save.

Repeat for every MCP tool connection.

## 10) Verify each connection in the Copilot Studio UI

In Copilot Studio:

1. Open each MCP tool connection.
2. Sign in.
3. Confirm connection status is healthy/connected.
4. Confirm the tool appears as available in the agent tool list.

## 11) Finalize agent behavior

In Copilot Studio:

1. Add instruction text that tells the agent when to use each MCP tool.
2. Add any other knowledge sources (for example SharePoint) if needed.
3. Publish the agent.
4. Validate in your target channel (for example M365 Copilot Chat).

## Troubleshooting

- Connection fails during sign-in:
  - Check Redirect URI exists in Entra app registration.
  - Check client secret is valid and not expired.

- Copilot Studio shows a generic truncation error when you select Create:
  - This is usually a wizard-side length or validation issue.
  - Shorten the Server name and Server description.
  - Make sure the Authorization URL, Token URL, and Refresh URL all use the same tenant ID.
  - Try saving with only https://api.fabric.microsoft.com/DataAgent.Execute.All in Scopes first.

- Permission or consent errors:
  - Confirm required delegated permissions are present.
  - Confirm admin consent is granted.

- Tool connects but does not return data:
  - Confirm Data Agent is published.
  - Confirm MCP URL is correct for the right workspace and agent.

- Wrong tool selected by the agent:
  - Improve tool names and descriptions.
  - Add clearer routing instructions in agent instructions.
