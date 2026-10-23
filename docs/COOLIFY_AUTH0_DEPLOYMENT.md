# ClearFlow remote SurrealDB MCP (Coolify + Auth0)

This deployment exposes the repository's MCP tools over Streamable HTTP at
`https://surreal-mcp.duckdns.org/mcp`. Auth0 protects the MCP endpoint. The MCP
service connects to SurrealDB separately using a dedicated database-level user.

## Security boundary

- Do not use the SurrealDB root account in this service.
- Create a dedicated SurrealDB user scoped to the intended namespace/database with
  the database-level owner/admin permissions required for development. Do not grant
  server-level access or user-management privileges unless a reviewed task requires it.
- The tool set includes arbitrary SurrealQL and destructive CRUD operations. This is
  intentional for the requested development-admin scope, so treat OAuth access as
  privileged access.
- Keep Auth0 client secret and SurrealDB password in Coolify secret environment
  variables only. Never put real credentials in Git, images, logs, or this document.
- Use the Coolify private network for SurrealDB-to-MCP traffic if possible. Do not
  publish SurrealDB's port just for the MCP service.

## Auth0 setup

1. Create an Auth0 API (resource server) with identifier/audience
   `https://surreal-mcp.duckdns.org`. Use RS256 signing.
2. Create an Auth0 application for the server's OAuth proxy integration. Keep the
   client ID and client secret in Coolify. Enable Authorization Code and Refresh
   Token grants if available for the selected application type.
3. Configure the exact callback URL documented by the installed FastMCP Auth0
   provider version. For the default `Auth0Provider` callback this is
   `https://surreal-mcp.duckdns.org/auth/callback`. The value must match exactly.
4. Set `AUTH0_CONFIG_URL` to
   `https://YOUR_AUTH0_DOMAIN/.well-known/openid-configuration`,
   `AUTH0_CLIENT_ID`, `AUTH0_CLIENT_SECRET`, `AUTH0_AUDIENCE`, and
   `MCP_BASE_URL` in Coolify.
5. Verify Auth0's API audience/resource configuration matches the audience used by
   FastMCP. Auth0 may require its Resource Parameter Compatibility Profile for MCP
   clients that use the RFC 8707 `resource` parameter.

Do not configure ChatGPT until OAuth discovery, browser authorization, callback,
token issuance, and protected MCP calls pass in a staging test.

## Coolify configuration

- Build pack: Dockerfile.
- Domain: `https://surreal-mcp.duckdns.org`; TLS handled by Coolify proxy.
- Container port: `8000`; do not publish a host port directly.
- Set `MCP_TRANSPORT=http`, `MCP_HOST=0.0.0.0`, `MCP_PORT=8000`,
  `MCP_BASE_URL=https://surreal-mcp.duckdns.org`.
- Set `SURREAL_URL` to the internal SurrealDB service address plus `/rpc`
  when the services share a Coolify network (preferred). Otherwise use the
  database's secure `wss://...` endpoint.
- Set `SURREAL_NAMESPACE` to the namespace actually chosen in SurrealDB Studio,
  and `SURREAL_DATABASE=clearflow`.
- Set the database credentials and all Auth0 credentials as Coolify secrets.

The container refuses to start in HTTP mode when the required Auth0 settings are
missing. Local STDIO mode remains available for local MCP clients.

## Verification checklist

1. Container starts and connects to SurrealDB with the dedicated database user.
2. Unauthenticated `POST /mcp` is rejected with an OAuth challenge (not an anonymous
   tool invocation).
3. OAuth discovery endpoints return valid metadata.
4. Browser sign-in and callback complete successfully.
5. Authenticated `tools/list` works.
6. Authenticated read-only query succeeds against the `clearflow` database.
7. A controlled write/read/delete test runs only in a disposable test table.
8. Requests with missing, expired, wrong-audience, or invalid tokens are rejected.
9. Logs do not contain access tokens, client secrets, or database passwords.

## Status

This guide is a deployment plan, not a claim that the branch has been deployed or
end-to-end tested. Verify the exact FastMCP/Auth0 provider API against the resolved
dependency version before production use.
