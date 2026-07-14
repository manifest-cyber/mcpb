# Manifest Cyber MCP Integration

This repository distributes the Manifest Cyber MCP integration, exposing Manifest Cyber platform data (assets, components, vulnerabilities, products, SBOMs, labels) as MCP tools inside your AI assistant. Every install path connects to the same remote Manifest Cyber MCP server; pick the one for your client:

| Client | Install path |
|---|---|
| Claude Code | [Plugin marketplace hosted in this repo](#install-in-claude-code) |
| Codex | [Manual `config.toml` entry](#install-in-codex) |
| Claude Desktop | [`.mcpb` bundle from each release](#install-in-claude-desktop) |
| Other MCP clients | [Streamable HTTP endpoint + API key](#other-mcp-clients) |

All paths require a Manifest Cyber API key with read permissions.

## How it works

The extension implements no tools of its own. It is a thin remote proxy: a minimal Node script, bundled in the `.mcpb` together with its pinned dependencies (the [MCP SDK](https://www.npmjs.com/package/@modelcontextprotocol/sdk) and `zod`), that relays MCP traffic from Claude Desktop to a remote Manifest Cyber MCP server. The remote server is where the tools live; it translates tool calls into Manifest Cyber platform API calls.

```
Claude Desktop
      |
      |  stdio (JSON-RPC over stdin/stdout)
      v
Bundled proxy (server/index.mjs inside the .mcpb)
      |
      |  Streamable HTTP
      v
Manifest Cyber MCP server
      |
      |  Manifest Cyber platform API calls
      v
Manifest Cyber platform
```

The bundled proxy exists only because Claude Desktop requires a local entry point. Claude Code, Codex, and other clients with native Streamable HTTP support connect to the Manifest Cyber MCP server directly; the proxy layer above does not apply to them.

Under the hood:

- Claude Desktop runs the extension's entry point in an Electron UtilityProcess using its built-in Node runtime. In that sandbox a spawned child process does not get a working stdio link to Claude Desktop's MCP channel, so the bridge cannot be launched as a subprocess. It therefore runs **in-process**.
- The proxy pumps JSON-RPC messages between two MCP SDK transports: `StdioServerTransport` (this process's own stdin/stdout, the channel Claude Desktop provides) and `StreamableHTTPClientTransport` (the remote Manifest Cyber MCP server).
- Requests are authenticated with the Manifest Cyber API key you set in the extension settings. The key needs read permissions.
- The proxy is stateless and stores nothing. Your API key is stored encrypted by Claude Desktop (macOS Keychain) and is only ever sent to the server URL you configure.
- Because the tools are defined server-side, tool additions and fixes reach you without an extension update. A new extension release is only needed when the proxy, its dependencies, or the manifest metadata change.

## Install in Claude Code

This repo is a [Claude Code plugin marketplace](https://code.claude.com/docs/en/plugin-marketplaces). In a Claude Code session:

```
/plugin marketplace add manifest-cyber/mcpb
/plugin install manifest-cyber@manifest-cyber
```

Or from the shell: `claude plugin marketplace add manifest-cyber/mcpb`, then `claude plugin install manifest-cyber@manifest-cyber`.

When the plugin is enabled, Claude Code prompts for your Manifest Cyber API key and stores it in secure storage (macOS Keychain). The plugin connects directly to the Manifest Cyber MCP server over Streamable HTTP; no local server or Node runtime is involved.

Notes:

- While this repository is private, installing requires GitHub access to it with working git credentials (`gh auth login` and `gh auth setup-git`, or SSH).
- To target a different server, set `MANIFEST_MCP_URL` to a full endpoint URL including the `/mcp` path before starting Claude Code. Unset, it defaults to the production server (`https://mcp.manifestcyber.com/mcp`).
- Update with `/plugin update manifest-cyber@manifest-cyber`, or turn on auto-update for the marketplace under **/plugin → Marketplaces**. Versions track commits to this repo.
- Verify with `/mcp`: the `manifest-cyber` server should show as connected.

## Install in Codex

Codex has no plugin marketplace; add the server to `~/.codex/config.toml` manually:

```toml
[mcp_servers.manifest]
url = "https://mcp.manifestcyber.com/mcp"
bearer_token_env_var = "MANIFEST_API_KEY"
```

Export your API key in the environment Codex runs from (e.g. `export MANIFEST_API_KEY=...` in your shell profile), then verify with `codex mcp list`. Note that `codex mcp add` only covers stdio servers; HTTP servers are configured in `config.toml` as above.

## Install in Claude Desktop

Requires Claude Desktop on macOS.

1. Download the `.mcpb` file from the latest [release](https://github.com/manifest-cyber/mcpb/releases).
2. In Claude Desktop: **Settings → Extensions**, install the downloaded `manifest-cyber-X.Y.Z.mcpb` (allow the unsigned extension).
3. Set the configuration fields, then enable the extension.

| Setting | Required | Notes |
|---|---|---|
| MCP server URL | Yes | Full MCP endpoint URL including the `/mcp` path. Defaults to the Manifest Cyber production server. |
| Manifest Cyber API key | Yes | Your Manifest Cyber API key with read permissions. Stored encrypted in the macOS Keychain. |

### Updating

Claude Desktop does not upgrade an installed extension in place. Remove the existing **Manifest Cyber** entry under **Settings → Extensions**, then install the new `.mcpb`.

## Other MCP clients

The Manifest Cyber MCP server is a standard remote MCP server (Streamable HTTP with bearer auth). Any client that supports remote servers with custom headers can connect to `https://mcp.manifestcyber.com/mcp` with the header `Authorization: Bearer <your API key>`. For example, in Claude Code without the plugin:

```
claude mcp add --transport http manifest https://mcp.manifestcyber.com/mcp --header "Authorization: Bearer YOUR_KEY"
```

## Allowlisting in Claude Desktop (Team / Enterprise)

Org Owners can restrict which extensions users may install via the desktop-extension allowlist (requires Claude Desktop >= 0.13.91):

1. Enable the allowlist under **Organization settings → Connectors → Desktop** (it is off by default; once on, all extensions are blocked except those you add).
2. Add the **Manifest Cyber** extension, uploaded as a custom extension if it is not in the public directory. Its machine `name` is `manifest-cyber` and its vendor (`author.name`) is `Manifest Cyber`; the `name` must be unique within the org.

Caveats:

- Machine-level MDM keys override the in-app allowlist. For the allowlist to populate, `isDesktopExtensionEnabled` and `isDesktopExtensionDirectoryEnabled` must not be set to `false` in your MDM profile (`com.anthropic.claudefordesktop` on macOS).
- `isDesktopExtensionSignatureRequired = true` rejects extensions not signed by a trusted publisher. This bundle is currently unsigned.

## Troubleshooting

- Per-server log: `~/Library/Logs/Claude/mcp-server-Manifest Cyber.log`. The proxy logs to stderr with a `[manifest-cyber]` prefix; a clean start shows no errors.
- **"No API key configured"**: set the API key (and URL) in the extension settings, then enable. With no key the proxy starts anyway, but every request fails with an authentication error.
- **`401` in the log**: the API key is invalid, revoked, or missing read permissions. Verify the key in the Manifest Cyber platform.
- **Invalid `MANIFEST_MCP_URL`**: the server URL setting must be a full URL including the `/mcp` path.

## License

UNLICENSED. Proprietary to Manifest Cyber.
