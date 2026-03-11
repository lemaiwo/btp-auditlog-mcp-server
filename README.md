# BTP Audit Log MCP Server

An MCP (Model Context Protocol) server for the SAP BTP Audit Log Retrieval API, powered by [odata-mcp-proxy](https://www.npmjs.com/package/odata-mcp-proxy). It exposes audit log records as MCP tools, allowing AI assistants like Claude to retrieve and analyze security events, configuration changes, data access, and data modification logs through natural language.

The entire server is defined through a single JSON config file -- no custom code required.

## How It Works

This project uses the `odata-mcp-proxy` npm package, which maps OData/REST services to MCP tools based on a configuration file. You provide a config describing your APIs and entity sets, and the proxy generates the corresponding MCP tools automatically.

```
AI Assistant (Claude, Cursor, etc.)
        |
        | MCP Protocol (HTTP or stdio)
        v
  odata-mcp-proxy
        |
        | REST + OAuth2 (via BTP Destination Service)
        v
  BTP Audit Log Retrieval API (/auditlog/v2)
```

Think of it like the [SAP Application Router](https://www.npmjs.com/package/@sap/approuter) -- a ready-made runtime you configure, not code you write.

## Exposed API

The config file (`btp-audit-log-api-config.json`) defines the Audit Log Retrieval API:

| Tool | Operations | Description |
|------|-----------|-------------|
| `AuditLogRecords` | list | Retrieve security events, configuration changes, data access, and data modification logs |

Results use server-side paging with a page size of 500 records. Use the `handle` parameter from the response to fetch the next page.

The `_list` tool supports OData query parameters: `$filter`, `$select`, `$expand`, `$orderby`, `$top`, `$skip`.

### Example queries

- List recent audit log entries
- Filter audit logs by time range
- Find security-related events
- Review configuration changes made in the last 24 hours

## Prerequisites

- **Node.js** 18+ (20+ recommended)
- **SAP BTP account** with a Cloud Foundry environment
- **BTP Destination** named `AUDIT_LOG_SERVICE` pointing to the Audit Log Retrieval API with OAuth2 authentication
- **Cloud Foundry CLI** (`cf`) and **MBT Build Tool** (`mbt`) for deployment

## Project Structure

```
btp-auditlog-mcp-server/
├── package.json                       # Start script + odata-mcp-proxy dependency
├── btp-audit-log-api-config.json      # API configuration (defines all MCP tools)
├── mta.yaml                           # BTP Cloud Foundry deployment descriptor
├── xs-security.json                   # XSUAA OAuth2 configuration
├── default-env.json                   # Local dev credentials (gitignored)
└── LICENSE
```

## Getting Started

### 1. Install dependencies

```bash
npm install
```

### 2. Configure BTP destination

Create a BTP Destination for the Audit Log Retrieval API:

| Destination | URL |
|-------------|-----|
| `AUDIT_LOG_SERVICE` | `https://auditlog-management.cfapps.<region>.hana.ondemand.com` |

The destination should use OAuth2 client credentials authentication.

### 3. Local development

Create a `default-env.json` with your BTP service bindings (XSUAA, Destination, Connectivity) to run locally:

```bash
npm start
```

This runs `odata-mcp-proxy --config btp-audit-log-api-config.json`.

### 4. Deploy to BTP

```bash
npm run build:btp     # Build MTA archive
npm run deploy:btp    # Deploy to Cloud Foundry
```

The MTA deployment provisions three service instances:
- **Destination** (lite) -- resolves the Audit Log API endpoint and manages OAuth2 tokens
- **Connectivity** (lite) -- enables secure backend connectivity
- **XSUAA** (application) -- handles OAuth2 authentication with role-based access control

## Security

The XSUAA configuration (`xs-security.json`) defines three role templates:

| Role | Scopes | Description |
|------|--------|-------------|
| `MCPViewer` | read | Read-only access to audit logs |
| `MCPEditor` | read, write | Read and write access |
| `MCPAdmin` | read, write, admin | Full administrative access |

OAuth2 redirect URIs are pre-configured for Claude.ai, Cursor, Microsoft Teams, and local development.

## Creating Your Own MCP Server

This project demonstrates how easy it is to create a custom MCP server using `odata-mcp-proxy`. To build your own:

1. Create a new project and install the dependency:
   ```bash
   mkdir my-mcp-server && cd my-mcp-server
   npm init -y
   npm install odata-mcp-proxy
   ```

2. Add a start script to `package.json`:
   ```json
   {
     "scripts": {
       "start": "odata-mcp-proxy --config my-api-config.json"
     }
   }
   ```

3. Define your APIs in a config file (`my-api-config.json`):
   ```json
   {
     "server": {
       "name": "my-mcp-server",
       "version": "1.0.0",
       "description": "My custom MCP server"
     },
     "apis": [
       {
         "name": "my-api",
         "destination": "MY_BTP_DESTINATION",
         "pathPrefix": "/api/v1",
         "csrfProtected": true,
         "entitySets": [
           {
             "entitySet": "Products",
             "description": "Product catalog",
             "category": "master-data",
             "keys": [{ "name": "Id", "type": "string" }],
             "operations": { "list": true, "get": true, "create": false, "update": false, "delete": false }
           }
         ]
       }
     ]
   }
   ```

4. Add your `mta.yaml`, `xs-security.json`, and BTP Destinations, then deploy. That's it -- no code to write.

## License

MIT
