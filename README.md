# Copilot Studio Agent -- Fabric Data Agent MCP

This project automates the setup of Microsoft Fabric Data Agents as MCP (Model Context Protocol) endpoints and connects them to a Copilot Studio agent via OAuth 2.0.

The updated setup flow treats Azure Key Vault plus a Power Platform Secret environment variable as the operational home for the OAuth client secret, instead of treating the raw secret value as the final destination.

## What it does

1. Discovers Fabric workspaces and Data Agents you have access to.
2. Validates the MCP endpoint URL for each agent.
3. Creates an Entra app registration with the required Fabric API permissions.
4. Generates or rotates the Entra client secret used for OAuth.
5. Stores that secret in Azure Key Vault.
6. Prints the metadata needed to create a Secret environment variable in the Copilot Studio solution that references the Key Vault secret.
7. Grants admin consent on the Fabric scopes.
8. Validates an OAuth token against the Fabric API.
9. Outputs the connection values you need to configure the MCP tool in Copilot Studio.

## Files

- `setup_fabric_mcp_copilot_studio.ipynb`: Main notebook. It now includes Azure Key Vault storage guidance and prints the solution Secret environment variable metadata.
- `MCP_COPILOT_STUDIO_SETUP_PLAIN_LANGUAGE.md`: Browser-only runbook for admins and makers who are not running the notebook.

## Prerequisites

- Python 3.9+
- Azure CLI (`az`) installed and signed in (`az login`)
- A Microsoft Fabric capacity with at least one published Data Agent
- Permissions to create Entra app registrations and grant admin consent
- Access to an Azure Key Vault in the same tenant as the Power Platform environment, or an Azure admin who manages it
- Permission to create environment variables in the Power Platform solution that contains the Copilot Studio agent

## Getting started

1. Clone this repo.
2. Create and activate a virtual environment:

   ```powershell
   python -m venv .venv
   .venv\Scripts\activate
   ```

3. Install dependencies (the notebook's first cell lists them).
4. Open `setup_fabric_mcp_copilot_studio.ipynb` and run the cells in order.
5. In the notebook, use Step 7A to write the app secret into Azure Key Vault and capture the exact values for the Secret environment variable in your Copilot Studio solution.
6. Use the generated OAuth and MCP settings to finish the tool connection in Copilot Studio.

If you prefer not to run the notebook, follow the steps in `MCP_COPILOT_STUDIO_SETUP_PLAIN_LANGUAGE.md` using only browser portals.

## Notes

- Set `WORKSPACE_IDS = []` in Step 1 to discover all workspaces automatically, or provide a list of specific workspace GUIDs to restrict the scan.
- Microsoft documents Azure Key Vault-backed Secret environment variables for Power Platform and Copilot Studio. The MCP connection itself is still created interactively in Copilot Studio, so verify in your tenant whether the OAuth blade exposes direct environment variable binding or whether you need to retrieve the secret from Key Vault during connection creation.
