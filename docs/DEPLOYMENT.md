# Deployment Guide

## Prerequisites

1. SAP BTP Global Account with CloudFoundry environment
2. Cloud Foundry CLI installed and configured
3. MBT (Multi-Target Application Archive Builder) installed
4. Access to on-premise SAP system
5. SAP Cloud Connector configured (for on-premise connectivity)

## Step-by-Step Deployment

### 1. Prepare BTP Environment

```bash
# Login to Cloud Foundry
cf login -a https://api.cf.{region}.hana.ondemand.com

# Target your org and space
cf target -o your-org -s your-space
```

### 2. Configure Destination in BTP

You must create a destination in SAP BTP to connect to your on-premise SAP system. This destination should use **Basic Authentication** and the **virtual hostname** configured in SAP Cloud Connector as the URL.

#### Option 1: Use the default destination name

- Create a destination in BTP with the name `SAP_SYSTEM`.

#### Option 2: Use a custom destination name

- Create a destination in BTP with a name of your choice.
- Set the environment variable `SAP_DESTINATION_NAME` to your chosen destination name when deploying the application.

#### Example Destination Configuration

- **Name:** SAP_SYSTEM (or your custom name)
- **Type:** HTTP
- **Authentication:** BasicAuthentication
- **Proxy Type:** OnPremise
- **User:** [your_sap_username]
- **Password:** [your_sap_password]
- **URL:** https://[virtual-hostname]:[port] (as configured in SAP Cloud Connector)

![Example Destination Configuration](./img/destination.png)

For more details on creating destinations, see the [SAP BTP documentation](https://help.sap.com/docs/btp/sap-business-technology-platform/creating-destinations).

### 3. Build the Application

Use the following npm script to build the application and generate the MTAR archive:

```bash
npm run build:btp
```

This will compile the project and create the MTAR file in the `mta_archives` directory.

### 4. Deploy the Application

Use the following npm script to deploy the MTAR archive to SAP BTP:

```bash
npm run deploy:btp
```

This will upload and deploy the application to your Cloud Foundry space.

### 5. Update XSUAA for VSCode MCP Integration (Optional)

If you plan to use VSCode MCP integration with OAuth authentication:

#### 5.1. Verify xs-security.json Configuration

The `xs-security.json` file already includes VSCode-compatible redirect URIs:

```json
{
  "oauth2-configuration": {
    "redirect-uris": [
      "https://*.cfapps.*.hana.ondemand.com/**",
      "http://localhost:3000/oauth/callback",
      "http://localhost:6274/**"
    ]
  }
}
```

**Critical Requirements:**
- ✅ Use `localhost` NOT `127.0.0.1` (XSUAA requirement)
- ✅ Include fixed port (`http://localhost:3000/oauth/callback`) for standard VSCode setup
- ✅ Include wildcard pattern (`http://localhost:6274/**`) for dynamic port allocation
- ✅ BTP wildcard (`https://*.cfapps.*.hana.ondemand.com/**`) for hosted callback

#### 5.2. Update XSUAA Service

After initial deployment, if you need to update XSUAA configuration:

```bash
# Get your XSUAA service instance name
cf services | grep xsuaa

# Update the XSUAA service with xs-security.json
cf update-service <YOUR_XSUAA_SERVICE_NAME> -c xs-security.json

# Wait for update to complete (check status)
cf service <YOUR_XSUAA_SERVICE_NAME>

# Restage the application to apply changes
cf restage btp-sap-odata-to-mcp-server
```

#### 5.3. Verify OAuth Configuration

Test the OAuth endpoints after deployment:

```bash
# Replace with your actual BTP app URL
export BTP_APP_URL="https://your-app.cfapps.eu10.hana.ondemand.com"

# Test OAuth discovery endpoint
curl $BTP_APP_URL/.well-known/oauth-authorization-server | jq

# Test OAuth health check
curl $BTP_APP_URL/oauth/health | jq

# Test client registration endpoint
curl $BTP_APP_URL/oauth/client-registration | jq
```

Expected OAuth health response:
```json
{
  "status": "configured",
  "oauth": {
    "configured": true,
    "capabilities": {
      "pkce": true,
      "refresh_tokens": true
    }
  },
  "vscode_compatibility": {
    "redirect_uris_supported": [
      "https://your-app.cfapps.eu10.hana.ondemand.com/oauth/callback",
      "http://localhost:3000/oauth/callback",
      "http://localhost:6274/**"
    ]
  }
}
```

#### 5.4. Get Client Credentials for VSCode

```bash
# View XSUAA credentials from environment
cf env btp-sap-odata-to-mcp-server | grep -A 20 xsuaa

# Or use the client registration endpoint (includes client_secret)
curl $BTP_APP_URL/oauth/client-registration | jq
```

Copy the `client_id` and `client_secret` for your VSCode MCP configuration.

### 6. Configure VSCode MCP Client

Create or update your VSCode MCP settings:

1. Copy `.vscode/settings.example.json` to `.vscode/settings.json`
2. Update with your BTP app URL and XSUAA credentials:

```json
{
  "mcp.servers": {
    "sap-odata-btp": {
      "type": "http",
      "url": "https://your-app.cfapps.eu10.hana.ondemand.com/mcp",
      "transport": "streamable-http",
      "authentication": {
        "type": "oauth2",
        "flow": "authorization_code",
        "discoveryUrl": "https://your-app.cfapps.eu10.hana.ondemand.com/.well-known/oauth-authorization-server",
        "clientId": "sb-btp-sap-odata-to-mcp-server-development!t12345",
        "clientSecret": "your-xsuaa-client-secret",
        "redirectUri": "http://localhost:3000/oauth/callback",
        "scopes": ["openid"],
        "pkce": {
          "enabled": true,
          "method": "S256"
        }
      }
    }
  }
}
```

3. Restart VSCode to apply the configuration
4. The MCP extension should prompt you to authenticate via OAuth

### 7. Troubleshooting OAuth Issues

**Error: "Invalid redirect_uri"**
- Verify redirect URI matches exactly in xs-security.json
- Use `localhost` not `127.0.0.1`
- Update XSUAA service and restage app

**Error: "redirect_uri_mismatch"**
- Check for typos in redirect URI
- Ensure port number matches (3000 or 6274)
- Verify path is `/oauth/callback`

**OAuth works in Claude Desktop but not VSCode**
- Different redirect URIs are used
- Add VSCode-specific URI to xs-security.json
- Both can coexist in configuration

For detailed troubleshooting, see [XSUAA_VSCODE_REQUIREMENTS.md](./XSUAA_VSCODE_REQUIREMENTS.md).