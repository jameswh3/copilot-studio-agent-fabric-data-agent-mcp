# Copilot Studio Agent -- Fabric Data Agent MCP

This project automates the setup of Microsoft Fabric Data Agents as MCP (Model Context Protocol) endpoints and connects them to a Copilot Studio agent via OAuth 2.0.

## What it does

1. Discovers Fabric workspaces and Data Agents you have access to.
2. Validates the MCP endpoint URL for each agent.
3. Creates an Entra app registration with the required Fabric API permissions.
4. Generates a client secret for OAuth.
5. Grants admin consent on the Fabric scopes.
6. Validates an OAuth token against the Fabric API.
7. Outputs the connection values you need to configure the MCP tool in Copilot Studio.

## Files

| File | Description |
|------|-------------|
| `setup_fabric_mcp_copilot_studio.ipynb` | Main notebook -- run this to automate the end-to-end setup. |
| `MCP_COPILOT_STUDIO_SETUP_PLAIN_LANGUAGE.md` | Browser-only runbook for admins and makers who are not running the notebook. |

## Prerequisites

- Python 3.9+
- Azure CLI (`az`) installed and signed in (`az login`)
- A Microsoft Fabric capacity with at least one published Data Agent
- Permissions to create Entra app registrations and grant admin consent

## Getting started

1. Clone this repo.
2. Create and activate a virtual environment:
   ```
   python -m venv .venv
   .venv\Scripts\activate
   ```
3. Install dependencies (the notebook's first cell lists them).
4. Open `setup_fabric_mcp_copilot_studio.ipynb` and run the cells in order.
5. Use the output values to configure the MCP tool connection in Copilot Studio.

If you prefer not to run the notebook, follow the steps in `MCP_COPILOT_STUDIO_SETUP_PLAIN_LANGUAGE.md` using only browser portals.

## Notes

- Set `WORKSPACE_IDS = []` in Step 1 to discover all workspaces automatically, or provide a list of specific workspace GUIDs to restrict the scan.
