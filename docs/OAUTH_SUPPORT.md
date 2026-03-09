# Model Context Protocol OAuth Authorization

This document describes the OAuth 2.1 authorization implementation for Model Context Protocol (MCP), following the [MCP 2025-03-26 Authorization Specification](https://modelcontextprotocol.io/specification/2025-03-26/basic/authorization/).

## Features

- Full support for OAuth 2.1 authorization flow
- PKCE support for enhanced security
- Authorization server metadata discovery
- Dynamic client registration
- Automatic token refresh
- Authorized HTTP Client implementation

## Usage Guide

### 1. Enable Features

Enable the auth feature in Cargo.toml:

```toml
[dependencies]
rmcp = { version = "0.1", features = ["auth", "transport-streamable-http-client-reqwest"] }
```

### 2. Use OAuthState

```rust ignore
    // Initialize oauth state machine
    let mut oauth_state = OAuthState::new(&server_url, None)
        .await
        .context("Failed to initialize oauth state machine")?;
    oauth_state
        .start_authorization(&["mcp", "profile", "email"], MCP_REDIRECT_URI)
        .await
        .context("Failed to start authorization")?;

```

### 3. Get authorization url and do callback

```rust ignore
    // Get authorization URL and guide user to open it
    let auth_url = oauth_state.get_authorization_url().await?;
    println!("Please open the following URL in your browser for authorization:\n{}", auth_url);

    // Handle callback - In real applications, this is typically done in a callback server
    let auth_code = "Authorization code (`code` param) obtained from browser after user authorization";
    let csrf_token = "CSRF token (`state` param) obtained from browser after user authorization";
    let credentials = oauth_state.handle_callback(auth_code, csrf_token).await?;

    println!("Authorization successful, access token: {}", credentials.access_token);

```

### 4. Use Authorized Streamable HTTP Transport and create client

```rust ignore
    let am = oauth_state
        .into_authorization_manager()
        .ok_or_else(|| anyhow::anyhow!("Failed to get authorization manager"))?;
    let client = AuthClient::new(reqwest::Client::default(), am);
    let transport = StreamableHttpClientTransport::with_client(
        client,
        StreamableHttpClientTransportConfig::with_uri(MCP_SERVER_URL),
    );

    // Create client and connect to MCP server
    let client_service = ClientInfo::default();
    let client = client_service.serve(transport).await?;
```

### 5. Use Authorized HTTP Client after authorized

```rust ignore
    let client = oauth_state.to_authorized_http_client().await?;
```

## Complete Examples

- **Client**: `examples/clients/src/auth/oauth_client.rs`
- **Server**: `examples/servers/src/complex_auth_streamhttp.rs`

### Running the Examples

```bash
# Run the OAuth server
cargo run -p mcp-server-examples --example servers_complex_auth_streamhttp

# Run the OAuth client (in another terminal)
cargo run -p mcp-client-examples --example clients_oauth_client
```

## Authorization Flow Description

1. **Resource Metadata Discovery**: Client probes the server and extracts `WWW-Authenticate` parameters including `resource_metadata` URL and `scope`
2. **Protected Resource Metadata**: Client fetches resource server metadata (RFC 9728) to find authorization server(s) and supported scopes
3. **AS Metadata Discovery**: Client discovers authorization server metadata via RFC 8414 and OpenID Connect well-known endpoints
4. **Client Registration**: If supported, client dynamically registers itself (or uses URL-based Client ID via SEP-991)
5. **Scope Selection**: SDK picks scopes from WWW-Authenticate > PRM > AS metadata > caller defaults
6. **Authorization Request**: Build authorization URL with PKCE (S256) and RFC 8707 resource parameter
7. **Authorization Code Exchange**: After user authorization, exchange code for access token (with resource parameter)
8. **Token Usage**: Use access token for API calls via `AuthClient` or `AuthorizedHttpClient`
9. **Token Refresh**: Automatically use refresh token to get new access token when current one expires; previously granted scopes are forwarded in the refresh request so providers that require them (e.g. Azure AD v2) work correctly

## Security Considerations

- All tokens are securely stored in memory
- PKCE implementation prevents authorization code interception attacks
- Automatic token refresh support reduces user intervention
- Only accepts HTTPS connections or secure local callback URIs

## Troubleshooting

If you encounter authorization issues, check the following:

1. Ensure server supports OAuth 2.1 authorization
2. Verify callback URI matches server's allowed redirect URIs
3. Check network connection and firewall settings
4. Verify server supports metadata discovery or dynamic client registration

## References

- [MCP Authorization Specification](https://modelcontextprotocol.io/specification/2025-03-26/basic/authorization/)
- [OAuth 2.1 Specification Draft](https://oauth.net/2.1/)
- [RFC 8414: OAuth 2.0 Authorization Server Metadata](https://datatracker.ietf.org/doc/html/rfc8414)
- [RFC 7591: OAuth 2.0 Dynamic Client Registration Protocol](https://datatracker.ietf.org/doc/html/rfc7591)
- [RFC 8707: Resource Indicators for OAuth 2.0](https://datatracker.ietf.org/doc/html/rfc8707)
- [RFC 9728: OAuth 2.0 Protected Resource Metadata](https://datatracker.ietf.org/doc/html/rfc9728)
- [RFC 7636: Proof Key for Code Exchange (PKCE)](https://datatracker.ietf.org/doc/html/rfc7636)
- [RFC 6749 §6: Refreshing an Access Token](https://www.rfc-editor.org/rfc/rfc6749#section-6)
