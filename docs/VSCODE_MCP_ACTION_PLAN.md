# VSCode MCP Compatibility - Action Plan

## Overview

This action plan provides a step-by-step implementation guide for making the SAP BTP OData MCP Server fully compatible with VSCode's MCP integration. The plan is organized into phases based on priority and dependencies.

**Estimated Total Effort**: 16-24 hours for full implementation
**Target Completion**: Can be phased over 1-2 weeks

---

## Phase 1: Critical Compatibility Updates (Must Have)

**Goal**: Ensure basic VSCode MCP compatibility with proper tool annotations, metadata, and OAuth flow
**Estimated Effort**: 8-12 hours
**Priority**: HIGH

### Task 1.1: Add Tool Annotations with readOnlyHint

**File**: [src/tools/hierarchical-tool-registry.ts](src/tools/hierarchical-tool-registry.ts#L52-L114)

**Changes Required**:

#### Step 1: Update Tool Registration Interface

Verify that the MCP SDK supports annotations. If not using the latest SDK, update:

```bash
npm install @modelcontextprotocol/sdk@latest
```

#### Step 2: Add Annotations to Level 1 Tool (discover-sap-data)

**Location**: Line 56-70

**Current Code**:
```typescript
this.mcpServer.registerTool(
    "discover-sap-data",
    {
        title: "Level 1: Discover SAP Services and Entities",
        description: "[LEVEL 1 - DISCOVERY] Search for SAP services...",
        inputSchema: { /* ... */ }
    },
    handler
);
```

**Updated Code**:
```typescript
this.mcpServer.registerTool(
    "discover-sap-data",
    {
        title: "Discover SAP Services and Entities",
        description: "Search for SAP OData services and entities. Returns service IDs, names, and entity lists for selection. No authentication required.",
        annotations: {
            readOnlyHint: true,  // ✅ NEW: Skip confirmation dialog
            helpText: "Use this to explore available SAP services before executing operations."
        },
        inputSchema: {
            query: z.string().optional().describe("Search term for services or entities (e.g., 'customer', 'sales order'). Leave empty to list all services."),
            category: z.string().optional().describe("Filter by business area: business-partner, sales, finance, procurement, hr, logistics, or all (default)."),
            limit: z.number().min(1).max(50).optional().describe("Maximum results to return (default: 20).")
        }
    },
    async (args: Record<string, unknown>) => {
        return this.discoverServicesAndEntitiesMinimal(args);
    }
);
```

#### Step 3: Add Annotations to Level 2 Tool (get-entity-metadata)

**Location**: Line 72-86

**Updated Code**:
```typescript
this.mcpServer.registerTool(
    "get-entity-metadata",
    {
        title: "Get Entity Metadata",
        description: "Retrieve complete schema details for a specific SAP entity including properties, types, keys, and capabilities. No authentication required.",
        annotations: {
            readOnlyHint: true,  // ✅ NEW: Skip confirmation dialog
            helpText: "Call this after discovering entities to understand their structure before executing operations."
        },
        inputSchema: {
            serviceId: z.string().describe("Service ID from discovery results (use the 'serviceId' field)."),
            entityName: z.string().describe("Entity name from discovery results (use the 'entityName' field).")
        }
    },
    async (args: Record<string, unknown>) => {
        return this.getEntityMetadataFull(args);
    }
);
```

#### Step 4: Add Annotations to Level 3 Tool (execute-sap-operation)

**Location**: Line 88-111

**Updated Code**:
```typescript
this.mcpServer.registerTool(
    "execute-sap-operation",
    {
        title: "Execute SAP Operation",
        description: "Perform CRUD operations on SAP entities with authenticated user context. Supports read, create, update, and delete operations. Requires authentication.",
        annotations: {
            // ⚠️ NOTE: Do NOT add readOnlyHint: true here
            // VSCode should show confirmation dialog for write operations
            helpText: "Authentication required. Operations execute under your SAP identity with full audit trail."
        },
        inputSchema: {
            serviceId: z.string().describe("Service ID from discovery (use the 'id' field, NOT 'title')."),
            entityName: z.string().describe("Entity name from discovery (use the 'name' field, NOT 'entitySet')."),
            operation: z.enum(["read", "read-single", "create", "update", "delete"]).describe("Operation type to perform."),
            parameters: z.record(z.any()).optional().describe("Operation parameters (keys for read-single/update/delete, data for create/update)."),
            filterString: z.string().optional().describe("OData filter (without '$filter=' prefix). Example: \"Status eq 'Active'\""),
            selectString: z.string().optional().describe("OData select (without '$select=' prefix). Example: \"Name,Status,CreatedDate\". NOTE: Not all SAP APIs support $select. If operation fails, retry without this parameter."),
            expandString: z.string().optional().describe("OData expand (without '$expand=' prefix). Example: \"Customer,Items\""),
            orderbyString: z.string().optional().describe("OData orderby (without '$orderby=' prefix). Example: \"Name desc\""),
            topNumber: z.number().optional().describe("OData $top value (limit). Example: 10"),
            skipNumber: z.number().optional().describe("OData $skip value (offset for pagination). Example: 20"),
            useUserToken: z.boolean().optional().describe("Use authenticated user token (default: true for data operations).")
        }
    },
    async (args: Record<string, unknown>) => {
        return this.executeEntityOperation(args);
    }
);
```

**Verification Steps**:
1. Rebuild the project: `npm run build`
2. Test tool registration with MCP inspector: `npm run inspect:mcp`
3. Verify annotations appear in tool metadata
4. Test in VSCode to confirm confirmation dialogs behavior

---

### Task 1.2: Optimize Tool Descriptions for VSCode UI

**Goal**: Make tool descriptions concise and user-friendly for VSCode's tool picker

**Changes**:
- Remove implementation markers like `[LEVEL 1 - DISCOVERY]`
- Simplify descriptions to 1-2 sentences
- Move detailed instructions to `helpText` annotation
- Focus on user benefit, not internal architecture

**Already addressed in Task 1.1 code samples above.**

---

### Task 1.3: Validate Input Schema Descriptions

**Goal**: Ensure parameter descriptions are clear and concise for VSCode's parameter editor

**Review Checklist**:
- ✅ Each parameter has a clear description
- ✅ Descriptions explain the parameter's purpose, not just its type
- ✅ Examples are provided where helpful
- ✅ Constraints are documented (min/max, allowed values)

**Already addressed in Task 1.1 code samples above.**

---

### Task 1.4: Update MCP Server Capabilities Declaration

**File**: [src/mcp-server.ts](src/mcp-server.ts#L30-L33)

**Current Code**:
```typescript
this.mcpServer = new McpServer({
    name: "btp-sap-odata-to-mcp-server",
    version: "2.0.0"
});
```

**Recommended Enhancement**:
```typescript
this.mcpServer = new McpServer({
    name: "btp-sap-odata-to-mcp-server",
    version: "2.0.0",
    capabilities: {
        tools: {
            listChanged: true  // Notify clients when tool list changes
        },
        resources: {
            listChanged: true  // Notify clients when resource list changes
        },
        logging: {
            levels: ["error", "warn", "info", "debug"]
        }
    }
});
```

**Note**: Verify if SDK version supports capabilities declaration in constructor. If not, this is already handled in the HTTP endpoint response at [src/index.ts:208-212](src/index.ts#L208-L212).

---

### Task 1.5: Implement VSCode-Compatible OAuth Flow

**Goal**: Ensure OAuth authentication works seamlessly with VSCode MCP clients

**Priority**: CRITICAL - VSCode cannot authenticate without this

#### Problem Statement

VSCode MCP clients need to:
1. Discover OAuth endpoints automatically
2. Initiate OAuth flow programmatically
3. Handle redirect URIs for local development
4. Support PKCE (Proof Key for Code Exchange)
5. Store and refresh tokens securely

Current implementation works for browser-based flows but needs enhancements for VSCode extension scenarios.

---

#### Step 1: Enhance OAuth Discovery Metadata

**File**: [src/index.ts](src/index.ts#L413-L549)

**Current Issue**: Discovery metadata exists but may not include all VSCode-required fields.

**Required Changes**:

Add VSCode-specific redirect URIs to the discovery response:

```typescript
// Location: Line ~426
app.get(['/.well-known/oauth-authorization-server', '/.well-known/oauth-authorization-server/mcp'], (req, res) => {
    try {
        if (!authService.isConfigured()) {
            return res.status(501).json({
                error: 'OAuth not configured',
                message: 'XSUAA service is not configured for this deployment',
                setup_required: 'Bind XSUAA service to this application'
            });
        }

        const xsuaaMetadata = authService.getXSUAADiscoveryMetadata()!;
        const baseUrl = getBaseUrl(req);

        const discoveryMetadata = {
            // Core OAuth 2.0 Authorization Server Metadata
            issuer: xsuaaMetadata.issuer,
            authorization_endpoint: `${baseUrl}/oauth/authorize`,
            token_endpoint: `${baseUrl}/oauth/token`,
            userinfo_endpoint: `${baseUrl}/oauth/userinfo`,
            revocation_endpoint: `${baseUrl}/oauth/revoke`,
            introspection_endpoint: `${baseUrl}/oauth/introspect`,

            // ✅ ADD: VSCode-specific redirect URIs
            redirect_uris_supported: [
                `${baseUrl}/oauth/callback`,
                'http://127.0.0.1:*/oauth/callback',  // Local VSCode development
                'http://localhost:*/oauth/callback',   // Local VSCode development
                'vscode://callback',                    // VSCode custom scheme
                'vscode-insiders://callback'            // VSCode Insiders
            ],

            // Client Registration Endpoint (RFC 7591)
            registration_endpoint: `${baseUrl}/oauth/client-registration`,

            // Supported response types
            response_types_supported: [
                'code'
            ],

            // Supported grant types
            grant_types_supported: [
                'authorization_code',
                'refresh_token'
            ],

            // ✅ ADD: PKCE support (critical for VSCode)
            code_challenge_methods_supported: ['S256', 'plain'],

            // Client Registration Support
            registration_endpoint_auth_methods_supported: [
                'none'  // No authentication required for static client registration
            ],
            client_registration_types_supported: [
                'static'
            ],

            // ✅ ADD: VSCode-specific guidance
            'x-vscode-mcp': {
                recommended_flow: 'authorization_code_with_pkce',
                redirect_uri_template: `${baseUrl}/oauth/callback`,
                local_redirect_pattern: 'http://127.0.0.1:{port}/oauth/callback',
                requires_pkce: true,  // Recommend PKCE for security
                token_storage: 'Use VSCode SecretStorage API',
                refresh_strategy: 'Automatic refresh 5 minutes before expiry'
            },

            // Service documentation
            service_documentation: `${baseUrl}/docs`,

            // Additional XSUAA specific metadata
            'x-xsuaa-metadata': {
                client_id: xsuaaMetadata.clientId,
                identityZone: xsuaaMetadata.identityZone,
                tenantMode: xsuaaMetadata.tenantMode
            },

            // MCP-specific extensions
            'x-mcp-server': {
                name: 'btp-sap-odata-to-mcp-server',
                version: '2.0.0',
                mcp_endpoint: `${baseUrl}/mcp`,
                authentication_required: true,
                capabilities: [
                    'SAP OData service discovery',
                    'CRUD operations with JWT forwarding',
                    'Dual authentication model',
                    'Session-based MCP transport',
                    'Scope-based authorization'
                ]
            },

            // MCP Static Client Support
            'x-mcp-static-client': {
                supported: true,
                registration_endpoint: `${baseUrl}/oauth/client-registration`,
                client_id: xsuaaMetadata.clientId,
                client_authentication_method: 'client_secret_basic'
            }
        };

        res.setHeader('Access-Control-Allow-Origin', '*');
        res.setHeader('Cache-Control', 'public, max-age=3600');
        res.json(discoveryMetadata);
    } catch (error) {
        logger.error('Failed to generate OAuth discovery metadata:', error);
        res.status(500).json({
            error: 'Failed to generate discovery metadata',
            message: error instanceof Error ? error.message : 'Unknown error'
        });
    }
});
```

---

#### Step 2: Add PKCE Support to Authorization Endpoint

**File**: [src/index.ts](src/index.ts#L783-L836)

**Current Issue**: Authorization endpoint doesn't properly store PKCE parameters for VSCode flows.

**Required Changes**:

```typescript
// Location: Line ~783
app.get('/oauth/authorize', (req, res) => {
    logger.info(`Start OAuth authorization flow`);
    try {
        if (!authService.isConfigured()) {
            return res.status(501).json({
                error: 'OAuth not configured',
                message: 'XSUAA service is not configured for this deployment'
            });
        }

        const state = req.query.state as string || randomUUID();
        const baseUrl = getBaseUrl(req);
        const mcpRedirectUri = req.query.redirect_uri as string;

        // ✅ ADD: PKCE parameters (critical for VSCode)
        const mcpCodeChallenge = req.query.code_challenge as string;
        const mcpCodeChallengeMethod = req.query.code_challenge_method as string || 'S256';

        // ✅ ADD: VSCode client identification
        const clientId = req.query.client_id as string;
        const responseType = req.query.response_type as string || 'code';

        // Validate required parameters
        if (!mcpRedirectUri) {
            return res.status(400).json({
                error: 'invalid_request',
                error_description: 'Missing required parameter: redirect_uri',
                hint: 'VSCode MCP clients must provide a redirect_uri parameter'
            });
        }

        // ✅ ADD: Validate PKCE for VSCode clients
        if (!mcpCodeChallenge) {
            logger.warn('OAuth request without PKCE - not recommended for VSCode');
            // Don't reject, but log warning
        }

        // ✅ ADD: Validate redirect URI is allowed
        const allowedRedirectPatterns = [
            new RegExp(`^${baseUrl}/oauth/callback`),
            /^http:\/\/127\.0\.0\.1:\d+\/oauth\/callback$/,
            /^http:\/\/localhost:\d+\/oauth\/callback$/,
            /^vscode:\/\/callback/,
            /^vscode-insiders:\/\/callback/
        ];

        const isValidRedirect = allowedRedirectPatterns.some(pattern =>
            pattern.test(mcpRedirectUri)
        );

        if (!isValidRedirect) {
            return res.status(400).json({
                error: 'invalid_request',
                error_description: `Invalid redirect_uri: ${mcpRedirectUri}`,
                allowed_patterns: [
                    `${baseUrl}/oauth/callback`,
                    'http://127.0.0.1:{port}/oauth/callback',
                    'http://localhost:{port}/oauth/callback',
                    'vscode://callback',
                    'vscode-insiders://callback'
                ]
            });
        }

        const authUrl = authService.getAuthorizationUrl(state, baseUrl);

        // Store mapping in a simple in-memory store (you might want to use Redis in production)
        if (!globalThis.mcpProxyStates) {
            globalThis.mcpProxyStates = new Map();
        }

        globalThis.mcpProxyStates.set(state, {
            mcpRedirectUri,
            state,
            mcpCodeChallenge,
            mcpCodeChallengeMethod,
            clientId,  // ✅ NEW: Store client ID
            timestamp: Date.now()
        });

        // Clean up old states (older than 10 minutes)
        for (const [key, value] of globalThis.mcpProxyStates.entries()) {
            if (Date.now() - value.timestamp > 600000) {
                globalThis.mcpProxyStates.delete(key);
            }
        }

        logger.info(`MCP OAuth proxy initiated for redirect: ${mcpRedirectUri}`);
        logger.info(`PKCE: ${mcpCodeChallenge ? 'enabled' : 'disabled'}`);

        logger.info(`Authorize proxy redirecting to: ${authUrl}`);
        res.redirect(authUrl);
    } catch (error) {
        logger.error('Failed to initiate OAuth flow:', error);
        res.status(500).json({
            error: 'server_error',
            error_description: 'Failed to initiate OAuth flow',
            details: error instanceof Error ? error.message : 'Unknown error'
        });
    }
});
```

---

#### Step 3: Add PKCE Validation to Token Endpoint

**File**: [src/index.ts](src/index.ts#L838-L893)

**Required Changes**:

```typescript
// Location: Line ~838
const tokenHandler = async (req: Request, res: Response) => {
    logger.info(`Start OAuth token exchange flow - grant_type: ${req.body?.grant_type}`);
    const baseUrl = getBaseUrl(req);
    try {
        if (!authService.isConfigured()) {
            return res.status(501).json({
                error: 'oauth_not_configured',
                error_description: 'XSUAA service is not configured for this deployment'
            });
        }

        const grantType = req.body?.grant_type;
        let tokenData;

        if (grantType === 'authorization_code' || req.body?.code) {
            // Authorization code flow
            const code = req.body.code;
            const codeVerifier = req.body.code_verifier;  // ✅ NEW: PKCE verifier
            const redirectUri = req.body.redirect_uri;

            if (!code) {
                return res.status(400).json({
                    error: 'invalid_request',
                    error_description: 'Missing required parameter: code'
                });
            }

            // ✅ NEW: PKCE validation
            // Note: In a full implementation, you would:
            // 1. Store code_challenge with the authorization code
            // 2. Verify code_verifier matches the stored code_challenge
            // For now, we log and pass through to XSUAA
            if (codeVerifier) {
                logger.info('PKCE code_verifier provided - validating with XSUAA');
            }

            logger.info('Processing authorization_code grant');

            // ✅ ENHANCEMENT: Pass PKCE parameters to XSUAA if provided
            tokenData = await authService.exchangeCodeForToken(
                code,
                redirectUri || authService.getRedirectUri(baseUrl),
                codeVerifier  // Pass verifier to XSUAA
            );

        } else if (grantType === 'refresh_token' || req.body?.refresh_token) {
            // Refresh token flow
            const refreshToken = req.body.refresh_token;
            if (!refreshToken) {
                return res.status(400).json({
                    error: 'invalid_request',
                    error_description: 'Missing required parameter: refresh_token'
                });
            }
            logger.info('Processing refresh_token grant');
            tokenData = await authService.refreshAccessToken(refreshToken);
        } else {
            return res.status(400).json({
                error: 'unsupported_grant_type',
                error_description: 'Supported grant types: authorization_code, refresh_token'
            });
        }

        logger.info(`OAuth token exchange successful - grant_type: ${grantType}`);

        // ✅ ADD: Include additional metadata for VSCode clients
        const enhancedTokenData = {
            ...tokenData,
            token_type: tokenData.token_type || 'Bearer',
            expires_in: tokenData.expires_in,
            // ✅ NEW: VSCode-specific metadata
            'x-vscode-mcp': {
                server_endpoint: `${baseUrl}/mcp`,
                refresh_before_expiry: 300,  // Refresh 5 minutes before expiry
                session_management: 'automatic'
            }
        };

        res.json(enhancedTokenData);
    } catch (error) {
        logger.error('OAuth token exchange failed:', error);
        res.status(400).json({
            error: 'invalid_grant',
            error_description: error instanceof Error ? error.message : 'Token exchange failed'
        });
    }
};
```

---

#### Step 4: Update AuthService to Support PKCE

**File**: [src/services/auth-service.ts](src/services/auth-service.ts#L59-L94)

**Required Changes**:

```typescript
/**
 * Exchange authorization code for access token with optional PKCE support
 */
async exchangeCodeForToken(
    code: string,
    redirectUri?: string,
    codeVerifier?: string  // ✅ NEW: PKCE code verifier
): Promise<{ access_token: string; refresh_token?: string; expires_in: number; token_type?: string }> {
    if (!this.xsuaaCredentials) {
        throw new Error('XSUAA service not configured');
    }
    const creds = this.xsuaaCredentials as Record<string, string>;
    const tokenUrl = `${creds.url}/oauth/token`;

    const params: Record<string, string> = {
        grant_type: 'authorization_code',
        code,
        client_id: creds.clientid,
        client_secret: creds.clientsecret,
        redirect_uri: redirectUri || this.getRedirectUri()
    };

    // ✅ NEW: Add PKCE verifier if provided
    if (codeVerifier) {
        params.code_verifier = codeVerifier;
    }

    try {
        const response = await fetch(tokenUrl, {
            method: 'POST',
            headers: {
                'Content-Type': 'application/x-www-form-urlencoded',
                'Accept': 'application/json'
            },
            body: new URLSearchParams(params).toString()
        });

        if (!response.ok) {
            const errorText = await response.text();
            this.logger.error(`Token exchange failed: ${response.status} - ${errorText}`);
            throw new Error(`Token exchange failed: ${response.status} - ${errorText}`);
        }

        const tokenData = await response.json();
        return tokenData;
    } catch (error) {
        this.logger.error('Failed to exchange code for token:', error);
        throw error;
    }
}
```

---

#### Step 5: Create VSCode OAuth Configuration Example

**New File**: `docs/examples/vscode-mcp-config.json`

```json
{
  "mcpServers": {
    "sap-odata-btp": {
      "type": "http",
      "url": "https://your-btp-app.cfapps.eu10.hana.ondemand.com/mcp",
      "transport": "streamable-http",
      "authentication": {
        "type": "oauth2",
        "flow": "authorization_code",
        "discoveryUrl": "https://your-btp-app.cfapps.eu10.hana.ondemand.com/.well-known/oauth-authorization-server",
        "clientId": "your-client-id",
        "clientSecret": "your-client-secret",
        "redirectUri": "http://127.0.0.1:33418/oauth/callback",
        "scopes": ["openid", "profile", "email"],
        "pkce": {
          "enabled": true,
          "method": "S256"
        },
        "tokenStorage": "vscode-secrets",
        "autoRefresh": {
          "enabled": true,
          "beforeExpirySeconds": 300
        }
      },
      "metadata": {
        "name": "SAP OData MCP Server",
        "description": "Access SAP OData services via natural language",
        "version": "2.0.0"
      }
    }
  }
}
```

---

#### Step 6: Add OAuth Health Check Endpoint

**File**: [src/index.ts](src/index.ts)

Add new endpoint to verify OAuth configuration:

```typescript
// Add after /oauth/.well-known/oauth_metadata endpoint (around line 780)

// OAuth health check endpoint for VSCode debugging
app.get('/oauth/health', (req, res) => {
    try {
        const isConfigured = authService.isConfigured();
        const baseUrl = getBaseUrl(req);

        const health = {
            status: isConfigured ? 'configured' : 'not_configured',
            timestamp: new Date().toISOString(),
            oauth: {
                configured: isConfigured,
                endpoints: isConfigured ? {
                    discovery: `${baseUrl}/.well-known/oauth-authorization-server`,
                    authorize: `${baseUrl}/oauth/authorize`,
                    token: `${baseUrl}/oauth/token`,
                    callback: `${baseUrl}/oauth/callback`,
                    refresh: `${baseUrl}/oauth/refresh`,
                    userinfo: `${baseUrl}/oauth/userinfo`
                } : null,
                capabilities: {
                    pkce: true,
                    refresh_tokens: true,
                    client_registration: 'static',
                    code_challenge_methods: ['S256', 'plain']
                }
            },
            vscode_compatibility: {
                mcp_protocol: '2025-06-18',
                transport: ['stdio', 'streamable-http'],
                authentication_flows: ['authorization_code', 'authorization_code_with_pkce'],
                redirect_uris_supported: [
                    `${baseUrl}/oauth/callback`,
                    'http://127.0.0.1:*/oauth/callback',
                    'http://localhost:*/oauth/callback',
                    'vscode://callback',
                    'vscode-insiders://callback'
                ]
            }
        };

        res.json(health);
    } catch (error) {
        res.status(500).json({
            status: 'error',
            error: error instanceof Error ? error.message : 'Unknown error'
        });
    }
});
```

---

#### Step 7: Update Global Type Definitions

**File**: [src/index.ts](src/index.ts#L19-L28)

**Current Code**:
```typescript
declare global {
    var mcpProxyStates: Map<string, {
        mcpRedirectUri: string;
        state: string;
        mcpCodeChallenge?: string;
        mcpCodeChallengeMethod?: string;
        timestamp: number;
    }>;
}
```

**Updated Code**:
```typescript
declare global {
    var mcpProxyStates: Map<string, {
        mcpRedirectUri: string;
        state: string;
        mcpCodeChallenge?: string;
        mcpCodeChallengeMethod?: string;
        clientId?: string;           // ✅ NEW
        responseType?: string;        // ✅ NEW
        timestamp: number;
    }>;
}
```

---

#### Verification Steps for OAuth Flow

1. **Test OAuth Discovery**:
   ```bash
   curl https://your-app.cfapps.eu10.hana.ondemand.com/.well-known/oauth-authorization-server | jq
   ```
   - Verify `code_challenge_methods_supported` includes `['S256', 'plain']`
   - Verify `x-vscode-mcp` section is present

2. **Test OAuth Health Check**:
   ```bash
   curl https://your-app.cfapps.eu10.hana.ondemand.com/oauth/health | jq
   ```

3. **Test Authorization with PKCE**:
   ```bash
   # Generate PKCE challenge
   CODE_VERIFIER=$(openssl rand -base64 32 | tr -d "=+/" | cut -c1-43)
   CODE_CHALLENGE=$(echo -n $CODE_VERIFIER | openssl dgst -sha256 -binary | base64 | tr -d "=+/" | cut -c1-43)

   # Initiate authorization
   curl -v "https://your-app.cfapps.eu10.hana.ondemand.com/oauth/authorize?response_type=code&client_id=YOUR_CLIENT&redirect_uri=http://127.0.0.1:3000/callback&state=test123&code_challenge=$CODE_CHALLENGE&code_challenge_method=S256"
   ```

4. **Test Token Exchange with PKCE**:
   ```bash
   curl -X POST https://your-app.cfapps.eu10.hana.ondemand.com/oauth/token \
     -H "Content-Type: application/x-www-form-urlencoded" \
     -d "grant_type=authorization_code" \
     -d "code=AUTH_CODE_FROM_CALLBACK" \
     -d "redirect_uri=http://127.0.0.1:3000/callback" \
     -d "client_id=YOUR_CLIENT" \
     -d "code_verifier=$CODE_VERIFIER"
   ```

5. **Test with VSCode MCP Inspector**:
   - Configure VSCode MCP settings with OAuth
   - Verify OAuth flow completes successfully
   - Confirm token is stored and used in MCP requests

---

### Task 1.6: Testing Phase 1 Changes (Updated)

**Test Plan**:

1. **Build and Start Server**:
   ```bash
   npm run build
   npm run start:http
   ```

2. **Test OAuth Discovery**:
   ```bash
   # Test discovery endpoint
   curl https://localhost:3000/.well-known/oauth-authorization-server | jq

   # Test OAuth health
   curl https://localhost:3000/oauth/health | jq
   ```
   - Verify PKCE support is advertised
   - Verify VSCode-specific metadata is present

3. **Test with MCP Inspector**:
   ```bash
   npm run inspect:mcp
   ```
   - Verify tool annotations appear in inspector UI
   - Test each tool to confirm `readOnlyHint` behavior
   - Test OAuth flow completion

4. **Test OAuth Flow with PKCE**:
   - Follow PKCE test procedure from Task 1.5
   - Verify code_verifier validation works
   - Test token refresh flow

5. **Test with VSCode** (if available):
   - Install VSCode Insiders or latest stable with MCP support
   - Configure MCP server with OAuth settings
   - Complete OAuth authentication flow
   - Verify discovery tools don't show confirmation dialogs
   - Verify execution tool shows confirmation dialog for write operations
   - Test token refresh behavior

6. **Regression Testing**:
   - Verify Claude Desktop integration still works
   - Test all three levels of discovery
   - Test authenticated operations
   - Verify existing OAuth flows (browser-based) still work

**Acceptance Criteria**:
- ✅ Discovery tools (Level 1 & 2) execute without confirmation in VSCode
- ✅ Execution tool (Level 3) shows confirmation dialog for write operations
- ✅ Tool descriptions are clear and concise in VSCode UI
- ✅ OAuth discovery endpoint includes VSCode-specific metadata
- ✅ PKCE flow works end-to-end
- ✅ Token refresh works automatically
- ✅ OAuth health check endpoint provides diagnostic info
- ✅ Claude Desktop integration remains functional

---

## Phase 2: Documentation and Developer Experience (Should Have)

**Goal**: Provide comprehensive documentation for VSCode extension developers
**Estimated Effort**: 6-8 hours
**Priority**: MEDIUM

### Task 2.1: Create VSCode Integration Guide

**New File**: `docs/VSCODE_INTEGRATION_GUIDE.md`

**Contents**:
1. **Overview**: How the MCP server works with VSCode
2. **Prerequisites**: VSCode version, MCP extension requirements
3. **Installation Options**:
   - Direct HTTP connection to BTP deployment
   - Local development setup
4. **Authentication Setup**:
   - OAuth flow for VSCode extensions
   - Token management and refresh
   - Security considerations
5. **Configuration Examples**:
   - VSCode settings.json configuration
   - Environment variables
   - Service discovery configuration
6. **Usage Examples**:
   - Discovering SAP services
   - Reading entity data
   - Creating/updating entities
7. **Troubleshooting**:
   - Common issues and solutions
   - OAuth authentication problems
   - Network connectivity issues

**Template Structure**:
```markdown
# VSCode Integration Guide

## Quick Start

### 1. Configure VSCode MCP Extension

Add to `.vscode/settings.json`:
```json
{
  "mcp.servers": {
    "sap-odata": {
      "url": "https://your-btp-app.cfapps.eu10.hana.ondemand.com/mcp",
      "transport": "http",
      "authentication": {
        "type": "oauth2",
        "authorizationUrl": "https://your-btp-app.cfapps.eu10.hana.ondemand.com/oauth/authorize",
        "tokenUrl": "https://your-btp-app.cfapps.eu10.hana.ondemand.com/oauth/token",
        "clientId": "your-client-id"
      }
    }
  }
}
```

### 2. Authenticate

1. Open VSCode Command Palette (Ctrl+Shift+P)
2. Run "MCP: Authenticate Server"
3. Select "sap-odata"
4. Browser opens for SAP login
5. Complete authentication
6. Return to VSCode - you're ready!

### 3. Use SAP Tools in Copilot

Ask Copilot:
- "Show me available SAP customer services"
- "Read customer data from SAP"
- "Update customer email address"

[Continue with detailed sections...]
```

---

### Task 2.2: Document OAuth Integration Pattern

**New File**: `docs/OAUTH_INTEGRATION.md`

**Contents**:
1. **OAuth Architecture Overview**
2. **Discovery Endpoints**:
   - `/.well-known/oauth-authorization-server`
   - `/oauth/client-registration`
3. **Authorization Flow**:
   - Step-by-step sequence diagram
   - Request/response examples
4. **Token Management**:
   - Access token lifetime
   - Refresh token usage
   - Token storage recommendations
5. **Security Best Practices**:
   - Token storage (keychain/keytar)
   - PKCE for native apps
   - Token refresh strategies
6. **Code Examples**:
   - TypeScript/JavaScript examples for VSCode extensions
   - Token acquisition and refresh

---

### Task 2.3: Update README.md

**File**: [README.md](README.md)

**New Section**: Add VSCode integration section after Claude Desktop section

```markdown
## VSCode Integration

This MCP server is compatible with VSCode's MCP support (VSCode v1.99+).

### Quick Setup

1. **Install VSCode Insiders** or latest stable (v1.99+)
2. **Configure MCP Server** in `.vscode/settings.json`
3. **Authenticate** via OAuth flow
4. **Use with Copilot** for natural language SAP interactions

See [VSCode Integration Guide](./docs/VSCODE_INTEGRATION_GUIDE.md) for detailed instructions.

### Key Features for VSCode

- ✅ **Read-only Discovery**: No confirmation dialogs for browsing
- ✅ **Safe Operations**: Confirmation dialogs for write operations
- ✅ **OAuth Authentication**: Secure SAP BTP integration
- ✅ **Token Efficiency**: 3-level progressive discovery
- ✅ **Natural Language**: Ask Copilot in plain English

### Example VSCode Copilot Interactions

```
You: "What SAP services are available?"
Copilot: [Uses discover-sap-data tool, no confirmation needed]

You: "Show me the first 10 customers"
Copilot: [Uses get-entity-metadata, then execute-sap-operation with read]

You: "Update customer 12345's email to newemail@example.com"
Copilot: [Shows confirmation dialog with operation details]
You: [Review and confirm in VSCode UI]
Copilot: [Executes update operation]
```
```

---

### Task 2.4: Create Troubleshooting Guide

**New File**: `docs/TROUBLESHOOTING.md`

**Sections**:
1. **VSCode-Specific Issues**
2. **OAuth Authentication Problems**
3. **Network Connectivity Issues**
4. **SAP Backend Errors**
5. **Performance Issues**
6. **Common Misconfigurations**

---

## Phase 3: Enhanced Error Handling (Nice to Have)

**Goal**: Provide structured error responses for better VSCode UI integration
**Estimated Effort**: 4-6 hours
**Priority**: LOW

### Task 3.1: Create Structured Error Response Helper

**New File**: `src/utils/error-responses.ts`

```typescript
export interface StructuredError {
    error: string;
    message: string;
    details?: Record<string, unknown>;
    recovery?: string;
    code?: string;
}

export function createErrorResponse(
    error: unknown,
    context: {
        operation: string;
        entity?: string;
        service?: string;
    }
): { content: Array<{ type: "text"; text: string }>; isError: true } {

    const errorMessage = error instanceof Error ? error.message : String(error);

    const structuredError: StructuredError = {
        error: error instanceof Error ? error.name : "UnknownError",
        message: errorMessage,
        details: {
            operation: context.operation,
            entity: context.entity,
            service: context.service
        },
        code: extractErrorCode(errorMessage)
    };

    // Add recovery suggestions
    structuredError.recovery = getRecoverySuggestion(errorMessage, context);

    return {
        content: [{
            type: "text",
            text: JSON.stringify(structuredError, null, 2)
        }],
        isError: true
    };
}

function extractErrorCode(message: string): string | undefined {
    // Extract SAP error codes, HTTP status codes, etc.
    const httpMatch = message.match(/\b[45]\d{2}\b/);
    if (httpMatch) return `HTTP_${httpMatch[0]}`;

    const sapMatch = message.match(/\b[A-Z]{2}\d{3}\b/);
    if (sapMatch) return sapMatch[0];

    return undefined;
}

function getRecoverySuggestion(message: string, context: any): string {
    // Intelligent recovery suggestions based on error patterns
    if (message.toLowerCase().includes('not found')) {
        return `Use 'discover-sap-data' to find available ${context.entity ? 'entities' : 'services'}.`;
    }

    if (message.toLowerCase().includes('unauthorized') || message.toLowerCase().includes('authentication')) {
        return 'Please authenticate via OAuth flow and ensure your token is valid.';
    }

    if (message.toLowerCase().includes('$select')) {
        return `Retry the operation without the 'selectString' parameter. Many SAP APIs don't fully support $select.`;
    }

    return 'Review the error details and adjust your request parameters.';
}
```

### Task 3.2: Update Tool Handlers to Use Structured Errors

**File**: [src/tools/hierarchical-tool-registry.ts](src/tools/hierarchical-tool-registry.ts)

**Pattern**: Replace basic error responses with structured ones

**Example** (for `discoverServicesAndEntitiesMinimal` method):

```typescript
} catch (error) {
    this.logger.error('Error in Level 1 discovery:', error);
    return createErrorResponse(error, {
        operation: 'discover-sap-data',
        service: args.category as string
    });
}
```

---

## Phase 4: Advanced Features (Optional)

**Goal**: Implement advanced VSCode integration features
**Estimated Effort**: 8-12 hours
**Priority**: LOW

### Task 4.1: Dynamic Tool Registration Based on Authentication

**Concept**: Only expose `execute-sap-operation` tool when user is authenticated

**Implementation**:

**File**: [src/tools/hierarchical-tool-registry.ts](src/tools/hierarchical-tool-registry.ts)

```typescript
/**
 * Set the user's JWT token and update tool availability
 */
setUserToken(token?: string) {
    const wasAuthenticated = !!this.userToken;
    this.userToken = token;
    this.sapClient.setUserToken(token);

    const isAuthenticated = !!this.userToken;

    // If authentication status changed, update tool registration
    if (wasAuthenticated !== isAuthenticated) {
        this.updateToolRegistration();
    }

    this.logger.debug(`User token ${token ? 'set' : 'cleared'} for tool registry`);
}

/**
 * Update tool registration based on authentication status
 */
private updateToolRegistration() {
    if (this.userToken) {
        // Register Level 3 execution tool
        this.registerExecutionTool();
        this.logger.info('Registered execution tool - user authenticated');
    } else {
        // Unregister Level 3 execution tool (keep discovery tools)
        this.mcpServer.unregisterTool('execute-sap-operation');
        this.logger.info('Unregistered execution tool - user not authenticated');
    }

    // Notify clients that tool list changed
    this.mcpServer.notifyToolListChanged();
}
```

**Benefits**:
- Clearer UX: Users only see available tools
- Better security: Prevents unauthorized operation attempts
- VSCode integration: Tool picker updates dynamically

---

### Task 4.2: Implement Session Persistence

**Goal**: Store sessions in Redis or database for production scalability

**File**: Create `src/services/session-store.ts`

**Interface**:
```typescript
export interface SessionStore {
    get(sessionId: string): Promise<SessionData | null>;
    set(sessionId: string, data: SessionData, ttl: number): Promise<void>;
    delete(sessionId: string): Promise<void>;
    cleanup(): Promise<number>;
}

export interface SessionData {
    userToken?: string;
    userId?: string;
    createdAt: Date;
    lastAccessedAt: Date;
}
```

**Implementations**:
1. `MemorySessionStore` (current behavior)
2. `RedisSessionStore` (for production)
3. `FileSessionStore` (for single-instance deployments)

---

### Task 4.3: Create VSCode Extension Package

**Goal**: Package MCP server as VSCode extension for marketplace distribution

**Structure**:
```
vscode-extension/
├── package.json          # Extension manifest
├── src/
│   ├── extension.ts     # Extension entry point
│   ├── mcp-client.ts    # MCP client implementation
│   └── auth-provider.ts # OAuth authentication provider
├── README.md            # Extension documentation
└── CHANGELOG.md         # Version history
```

**Key Components**:

1. **Extension Activation**: Start MCP server as subprocess
2. **OAuth Provider**: Handle token acquisition and refresh
3. **Configuration UI**: Settings for SAP connection
4. **Status Bar**: Show connection status
5. **Commands**: Manual refresh, re-authenticate, etc.

**package.json excerpt**:
```json
{
  "name": "sap-odata-mcp",
  "displayName": "SAP OData MCP Server",
  "description": "Access SAP OData services via Model Context Protocol",
  "version": "1.0.0",
  "engines": {
    "vscode": "^1.99.0"
  },
  "categories": ["AI"],
  "activationEvents": ["onStartupFinished"],
  "contributes": {
    "configuration": {
      "title": "SAP OData MCP",
      "properties": {
        "sapOdataMcp.btpUrl": {
          "type": "string",
          "description": "SAP BTP application URL"
        }
      }
    }
  }
}
```

---

## Phase 5: Testing and Quality Assurance

**Goal**: Comprehensive testing across VSCode versions and scenarios
**Estimated Effort**: 6-8 hours
**Priority**: HIGH (before production release)

### Task 5.1: Unit Testing

**New Files**: Create tests in `src/__tests__/`

1. **Tool Registration Tests**:
   - Verify annotations are present
   - Validate input schemas
   - Test error handling

2. **Session Management Tests**:
   - Session creation and cleanup
   - Token association
   - Expiry handling

3. **OAuth Flow Tests**:
   - Authorization URL generation
   - Token exchange
   - Token refresh

### Task 5.2: Integration Testing

**Test Scenarios**:

1. **VSCode Integration Tests**:
   - Install and configure MCP server
   - Test OAuth authentication flow
   - Execute all three tool levels
   - Verify confirmation dialog behavior
   - Test error recovery

2. **Claude Desktop Compatibility Tests**:
   - Ensure Phase 1 changes don't break Claude Desktop
   - Test all existing workflows
   - Verify OAuth still works

3. **Load Testing**:
   - Concurrent sessions
   - Large result sets
   - Token refresh under load

### Task 5.3: User Acceptance Testing

**Test Plan**:

1. **Recruit Test Users**:
   - VSCode users familiar with SAP
   - Non-technical users for UX feedback

2. **Test Scenarios**:
   - First-time setup and authentication
   - Common SAP queries (read customers, orders)
   - Data modifications (create, update)
   - Error recovery (invalid entity, auth failure)

3. **Feedback Collection**:
   - Survey on ease of use
   - Documentation clarity
   - Feature requests

### Task 5.4: Performance Testing

**Metrics to Measure**:
- Tool execution latency
- Token efficiency (response sizes)
- Session overhead
- Memory usage under load

**Tools**:
- k6 or Artillery for load testing
- Chrome DevTools for network analysis
- VSCode performance profiler

---

## Implementation Timeline

### Week 1: Core Compatibility

| Day | Tasks | Deliverables |
|-----|-------|-------------|
| 1 | Task 1.1-1.3: Tool annotations and descriptions | Updated tool registry code |
| 2 | Task 1.4: Capabilities declaration | Updated MCP server initialization |
| 3 | Task 1.5: OAuth flow for VSCode (PKCE, discovery) | OAuth endpoints with PKCE support |
| 4 | Task 1.5 (cont): OAuth testing and validation | Verified OAuth flow |
| 5 | Task 1.6: Integration testing Phase 1 | All Phase 1 features verified |

### Week 2: Enhancements (Optional)

| Day | Tasks | Deliverables |
|-----|-------|-------------|
| 1-2 | Task 3.1-3.2: Structured errors | Enhanced error handling |
| 3-4 | Task 4.1: Dynamic tool registration | Advanced feature |
| 5 | Task 5.1-5.2: Testing | Test coverage |

---

## Verification Checklist

Before marking each phase complete, verify:

### Phase 1 Checklist
- [ ] All tools have appropriate `readOnlyHint` annotations
- [ ] Tool descriptions are concise and user-friendly
- [ ] Input schema descriptions are clear
- [ ] MCP Inspector shows annotations correctly
- [ ] OAuth discovery endpoint includes PKCE support
- [ ] OAuth discovery includes VSCode-specific metadata
- [ ] PKCE flow (authorization + token exchange) works end-to-end
- [ ] OAuth health check endpoint works
- [ ] Code verifier validation is implemented
- [ ] Redirect URI validation includes VSCode patterns
- [ ] VSCode configuration example is documented
- [ ] VSCode shows/hides confirmation dialogs as expected
- [ ] Token refresh works automatically
- [ ] Claude Desktop integration still works
- [ ] All unit tests pass

### Phase 2 Checklist
- [ ] VSCode integration guide is complete and accurate
- [ ] OAuth integration is fully documented
- [ ] README includes VSCode section
- [ ] Troubleshooting guide covers common issues
- [ ] Code examples work in VSCode extensions

### Phase 3 Checklist
- [ ] Error responses are structured and helpful
- [ ] Recovery suggestions are accurate
- [ ] Error codes are properly extracted
- [ ] VSCode UI displays errors clearly

### Phase 4 Checklist
- [ ] Dynamic tool registration works
- [ ] Session persistence is reliable
- [ ] VSCode extension package is functional
- [ ] Extension is published (if applicable)

### Phase 5 Checklist
- [ ] All unit tests pass (>80% coverage)
- [ ] Integration tests cover key scenarios
- [ ] UAT feedback is positive
- [ ] Performance metrics meet targets
- [ ] No regressions in Claude Desktop

---

## Risk Mitigation

### Risk 1: Breaking Claude Desktop Compatibility

**Mitigation**:
- Maintain backward compatibility with current API
- Test Claude Desktop after every change
- Use feature flags for VSCode-specific features
- Keep separate test suites for each client

### Risk 2: OAuth Complexity for Users

**Mitigation**:
- Provide clear step-by-step documentation
- Create video tutorials
- Offer example configurations
- Build VSCode extension to automate OAuth

### Risk 3: Performance Degradation

**Mitigation**:
- Benchmark before and after changes
- Monitor production metrics
- Implement caching where appropriate
- Load test with realistic scenarios

### Risk 4: SAP Connectivity Issues

**Mitigation**:
- Document SAP BTP prerequisites clearly
- Provide troubleshooting for common issues
- Test with multiple SAP backend versions
- Implement retry logic with exponential backoff

---

## Success Criteria

The VSCode MCP integration will be considered successful when:

1. **Functionality**:
   - ✅ All tools work in VSCode without errors
   - ✅ Discovery tools execute without confirmation dialogs
   - ✅ Execution tool shows confirmation for write operations
   - ✅ OAuth authentication completes successfully

2. **Documentation**:
   - ✅ Setup guide allows users to configure in <15 minutes
   - ✅ Common issues are documented with solutions
   - ✅ Code examples work without modification

3. **Compatibility**:
   - ✅ Works with VSCode v1.99+
   - ✅ Claude Desktop integration remains functional
   - ✅ No breaking changes to existing API

4. **User Experience**:
   - ✅ Tool descriptions are clear in VSCode picker
   - ✅ Error messages provide actionable guidance
   - ✅ Confirmation dialogs show relevant context
   - ✅ OAuth flow is smooth and well-explained

5. **Performance**:
   - ✅ Tool execution latency <2s for discovery
   - ✅ Tool execution latency <5s for data operations
   - ✅ No memory leaks in long-running sessions
   - ✅ Session management scales to 100+ concurrent users

---

## Rollout Strategy

### Development Environment
1. Implement Phase 1 changes in feature branch
2. Test with MCP Inspector
3. Test with VSCode locally
4. Merge to main after QA approval

### Staging Environment
1. Deploy to BTP staging space
2. Conduct integration testing
3. User acceptance testing with pilot users
4. Performance and load testing

### Production Environment
1. Deploy during maintenance window
2. Monitor error rates and performance
3. Gather user feedback
4. Iterate based on feedback

### Rollback Plan
- Keep previous version available as fallback
- Document rollback procedure
- Monitor for issues in first 48 hours
- Be prepared to rollback if critical issues arise

---

## Post-Implementation

### Monitoring
- Set up alerts for error rates
- Track tool usage metrics
- Monitor session creation/cleanup
- Track OAuth success rates

### Maintenance
- Update documentation based on user feedback
- Address bugs and issues
- Keep dependencies up to date
- Monitor VSCode MCP specification changes

### Future Enhancements
- Add prompt templates for common SAP tasks
- Implement sampling for AI-generated queries
- Support additional SAP authentication methods
- Add workspace-specific service filtering

---

## 📊 Effort Estimates (Updated)

- **Minimum viable VSCode compatibility**: 8-12 hours (Phase 1 with OAuth)
- **Full professional implementation**: 20-28 hours (Phases 1-3)
- **Complete with extension**: 28-40 hours (All phases)

### Effort Breakdown by Task
- Task 1.1-1.4: Tool annotations and metadata (4 hours)
- Task 1.5: OAuth flow implementation (4-6 hours)
- Task 1.6: Phase 1 testing (2 hours)
- Phase 2: Documentation (6-8 hours)
- Phase 3: Enhanced errors (4-6 hours)
- Phase 4: Advanced features (8-12 hours)
- Phase 5: Comprehensive testing (6-8 hours)

---

## Support and Contact

For questions or issues during implementation:
- GitHub Issues: [Repository URL]
- Documentation: [docs/ folder]
- SAP Community: [Link if applicable]

---

**Document Version**: 1.0
**Last Updated**: 2025
**Next Review**: After Phase 1 completion
