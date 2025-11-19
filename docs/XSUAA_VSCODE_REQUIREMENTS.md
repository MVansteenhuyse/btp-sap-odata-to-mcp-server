# XSUAA Requirements for VSCode MCP Integration

## Critical XSUAA Constraints

### 1. **localhost vs 127.0.0.1**
- ✅ **MUST use**: `http://localhost:PORT/callback`
- ❌ **NEVER use**: `http://127.0.0.1:PORT/callback`
- **Reason**: XSUAA (SAP's OAuth provider) only accepts `localhost`, not IP addresses

### 2. **Redirect URI Configuration**
All redirect URIs **MUST** be pre-configured in `xs-security.json`:

```json
{
  "oauth2-configuration": {
    "redirect-uris": [
      "https://*.cfapps.*.hana.ondemand.com/**",  // BTP wildcard
      "http://localhost:3000/oauth/callback",     // Fixed port
      "http://localhost:6274/**",                 // Dynamic port wildcard
      "vscode://callback"                         // Custom scheme (if needed)
    ]
  }
}
```

### 3. **XSUAA Wildcard Support**
XSUAA supports **specific wildcard patterns**:
- ✅ `http://localhost:6274/**` (port + path wildcard)
- ✅ `https://*.cfapps.*.hana.ondemand.com/**` (domain wildcard)
- ❌ `http://localhost:*/callback` (port wildcard NOT supported)

### 4. **BTP Hosting Model**

#### Current Architecture:
```
VSCode Extension (Local)
    ↓ HTTPS
SAP BTP Cloud Foundry (your-app.cfapps.eu10.hana.ondemand.com)
    ↓ OAuth
XSUAA (SAP's OAuth Server)
    ↓ Redirect
localhost:PORT/callback (VSCode Extension)
```

#### OAuth Flow:
1. VSCode extension initiates OAuth → `https://your-app.cfapps.eu10.hana.ondemand.com/oauth/authorize`
2. User redirected to XSUAA login
3. After login, XSUAA redirects to `http://localhost:PORT/callback`
4. VSCode extension captures code, exchanges for token
5. Token used for subsequent MCP requests

---

## VSCode MCP Configuration

### Required xs-security.json Updates

**Before using VSCode MCP, add the VSCode redirect URI:**

```json
{
  "oauth2-configuration": {
    "redirect-uris": [
      "https://*.cfapps.*.hana.ondemand.com/**",
      "http://localhost:3000/oauth/callback",    // ← Add this for VSCode
      "http://localhost:6274/**"                 // ← Or this for dynamic port
    ]
  }
}
```

**Deploy XSUAA service update:**
```bash
cf update-service YOUR_XSUAA_SERVICE -c xs-security.json
cf restage YOUR_APP_NAME
```

---

## VSCode Extension Configuration

### Example .vscode/settings.json

```json
{
  "mcp.servers": {
    "sap-odata-btp": {
      "type": "http",
      "url": "https://your-btp-app.cfapps.eu10.hana.ondemand.com/mcp",
      "transport": "streamable-http",
      "authentication": {
        "type": "oauth2",
        "flow": "authorization_code",
        "discoveryUrl": "https://your-btp-app.cfapps.eu10.hana.ondemand.com/.well-known/oauth-authorization-server",
        "clientId": "your-xsuaa-client-id",
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

**Important Notes:**
- `redirectUri` MUST use `localhost` (not 127.0.0.1)
- `redirectUri` MUST match entry in xs-security.json
- `clientId` and `clientSecret` from XSUAA service binding

---

## Troubleshooting

### Error: "Invalid redirect_uri"
**Cause**: Redirect URI not configured in XSUAA

**Solution**:
1. Add redirect URI to `xs-security.json`
2. Update XSUAA service: `cf update-service YOUR_XSUAA_SERVICE -c xs-security.json`
3. Restage app: `cf restage YOUR_APP_NAME`

### Error: "redirect_uri_mismatch"
**Cause**: Redirect URI doesn't exactly match xs-security.json

**Solution**:
- Check for `localhost` vs `127.0.0.1`
- Verify port number matches
- Ensure path matches (e.g., `/oauth/callback` vs `/callback`)

### OAuth flow works in Claude Desktop but not VSCode
**Cause**: Different redirect URI patterns

**Solution**:
- Claude Desktop may use different callback URL
- Add VSCode-specific redirect URI to xs-security.json
- Both can coexist in the same configuration

---

## Testing OAuth Configuration

### 1. Verify XSUAA Configuration
```bash
# Get XSUAA service details
cf service YOUR_XSUAA_SERVICE

# Check current redirect URIs
cf env YOUR_APP_NAME | grep -A 20 xsuaa
```

### 2. Test OAuth Discovery
```bash
curl https://your-app.cfapps.eu10.hana.ondemand.com/.well-known/oauth-authorization-server | jq '.redirect_uris_supported'
```

Expected output:
```json
[
  "https://your-app.cfapps.eu10.hana.ondemand.com/oauth/callback",
  "http://localhost:3000/oauth/callback",
  "http://localhost:6274/**"
]
```

### 3. Test OAuth Health
```bash
curl https://your-app.cfapps.eu10.hana.ondemand.com/oauth/health | jq
```

Should show:
```json
{
  "status": "configured",
  "oauth": {
    "configured": true,
    "capabilities": {
      "pkce": true
    }
  },
  "vscode_compatibility": {
    "redirect_uris_supported": [...]
  }
}
```

---

## Production Considerations

### Security Best Practices

1. **Restrict Redirect URIs**: Only add necessary redirect URIs to xs-security.json
2. **Use PKCE**: Always enable PKCE for VSCode extensions (already configured)
3. **Token Storage**: VSCode stores tokens in SecretStorage (secure)
4. **Token Expiry**: Configure appropriate token validity in xs-security.json

### Current xs-security.json Settings
```json
{
  "oauth2-configuration": {
    "token-validity": 3600,        // 1 hour access token
    "refresh-token-validity": 86400 // 24 hours refresh token
  }
}
```

### Scaling Considerations

For production deployments with many VSCode users:
- Consider using wildcard pattern: `http://localhost:6274/**`
- Allows dynamic port allocation
- More flexible for different VSCode setups
- Still secure (localhost-only)

---

## Key Differences: VSCode MCP vs Claude Desktop

| Aspect | Claude Desktop | VSCode MCP |
|--------|---------------|------------|
| Redirect URI | `https://claude.ai/api/mcp/auth_callback` | `http://localhost:PORT/callback` |
| OAuth Flow | Browser-based | Extension-based |
| Token Storage | Claude Desktop app | VSCode SecretStorage |
| PKCE Required | Recommended | Required |
| Configuration | `claude_desktop_config.json` | `.vscode/settings.json` |

---

## Additional Resources

- [XSUAA Documentation](https://help.sap.com/docs/BTP/65de2977205c403bbc107264b8eccf4b/51f2a5d9c6c441b8b2b52c50c2e6e94c.html)
- [SAP BTP Security Guide](https://help.sap.com/docs/BTP/65de2977205c403bbc107264b8eccf4b/e129aa20c78c4a9fb379b9803b02e5f6.html)
- [OAuth 2.0 PKCE](https://oauth.net/2/pkce/)
- [VSCode MCP Documentation](https://code.visualstudio.com/api/extension-guides/ai/mcp)

---

**Document Version**: 1.0
**Last Updated**: 2025
**Applies To**: SAP BTP Cloud Foundry deployments with XSUAA
