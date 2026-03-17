# Usandor o Google Workspace CLI dentro do Claude Cowork

## You don't need TCP/IP — MCP passthrough solves everything

**The core insight: Claude Cowork already bridges its VM sandbox to host-side MCP servers automatically.** You do not need TCP/IP, SSH tunnels, or HTTP wrappers. When you configure an MCP server in Claude Desktop, Cowork's Linux VM gains access to it via an internal SDK protocol passthrough over Unix pipes. The Google Workspace CLI (`gws`) supports MCP server mode over stdio — making it directly usable. Even better, mature community MCP servers for Google Drive and Sheets exist and require zero CLI wrapping.

## Cowork's sandbox is a full VM, but MCP punches through it

Claude Cowork is not a standalone product — it's a tab within Claude Desktop (alongside Chat and Code), launched January 12, 2026. It runs Claude Code CLI inside a **full Ubuntu 22.04 Linux VM** on Apple's Virtualization Framework, not merely a macOS App Sandbox. The VM gets **4 vCPUs, ~3.8 GB RAM, and ~10 GB disk** on your M3 Mac. Inside the VM, sessions are further isolated with bubblewrap and seccomp filters.

The sandbox restrictions are severe. Network access is limited to a strict allowlist: **only `api.anthropic.com`, `pypi.org`, and `registry.npmjs.org`** — all other domains return `403 Forbidden`. The VM cannot access host-installed CLI tools, Homebrew packages, or macOS-native utilities. `curl` to arbitrary URLs fails. This is why your TCP/IP instinct was reasonable.

But Anthropic built an escape hatch: **MCP servers configured in Claude Desktop are dynamically injected into the VM at session start** via the SDK protocol. The communication stack between host and VM includes dedicated channels for MCP (Unix pipes), HTTP proxy (Unix socket → socat), shared files (VirtioFS), and message passing. The MCP server process runs on the host with normal permissions — it can execute any host-installed CLI, access the full filesystem, and make arbitrary network requests. Cowork inside the VM calls these tools transparently.

## The Google Workspace CLI already speaks MCP

The `gws` CLI (at `github.com/googleworkspace/cli`) is a **Rust-based tool** that dynamically generates its command surface from Google's Discovery Service, covering **50+ Google Workspace APIs** including Drive, Sheets, Gmail, Calendar, Docs, and Chat. It launched March 2, 2026, and has already accumulated **~15,700 GitHub stars**. Install it via `npm install -g @googleworkspace/cli` — the npm package bundles pre-built ARM64 binaries for macOS.

The tool includes (or included) an MCP server mode activated with `gws mcp`. This starts a **stdio-based MCP server** that exposes Workspace APIs as structured tools. The MCP history is turbulent: the `mcp` subcommand was removed in v0.8.0 on March 7 citing context window overhead, but the current README on `main` still documents it and npm shows **v0.13.2** — well past the removal version. The MCP mode likely returned after community pushback. To configure it in Claude Desktop's `claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "gws": {
      "command": "gws",
      "args": ["mcp", "-s", "drive,sheets"]
    }
  }
}
```

This spawns `gws mcp` as a host-side subprocess. Cowork accesses it via the SDK passthrough — no TCP/IP involved. The `-s` flag filters to specific services (Drive and Sheets in this example), keeping the tool count manageable. Without filtering, `gws` can expose **200–400 tools** consuming 40,000–100,000 tokens of context, which is why the team temporarily removed MCP.

One practical caveat: `gws` requires OAuth credentials. Run `gws auth login` on your host terminal first to complete the interactive OAuth flow. Credentials are encrypted with AES-256-GCM and stored in the macOS keyring at `~/.config/gws/credentials.json`.

## Community MCP servers may be the better choice

If `gws mcp` proves unstable (the project warns of breaking changes before v1.0), several mature community MCP servers offer Google Drive and Sheets automation with battle-tested reliability:

**`taylorwilsdon/google_workspace_mcp`** is the most comprehensive option, covering **12 Google services with 100+ tools** — Drive, Sheets, Docs, Gmail, Calendar, Slides, Forms, Chat, Tasks, Contacts, Apps Script, and Search. It supports DXT one-click install for Claude Desktop, OAuth 2.1 with multi-user support, tool tiers to control context usage, and a read-only safety mode. Install via `uvx workspace-mcp --tool-tier core` or the DXT package.

**`xing5/mcp-google-sheets`** is purpose-built for Sheets-heavy workflows, exposing **19 dedicated tools** (~13K tokens) for full CRUD operations, batch updates, formatting, and sharing. It also includes Google Drive integration for file management. Supports service accounts for headless/automated use.

**`isaacphi/mcp-gdrive`** extends Anthropic's archived reference server with Sheets write support — a lighter option if you only need Drive search/read and Sheets read/write.

All three use stdio transport and work identically in Claude Desktop's config. Cowork picks them up automatically.

## Recommended architecture for your M3 Mac

The simplest production-ready setup requires no TCP/IP, no HTTP wrapping, and no SSH tunneling:

```
┌─────────────────────────────────────────────────┐
│  Claude Desktop (macOS M3)                      │
│  ┌──────────┐  ┌──────────┐  ┌───────────────┐ │
│  │   Chat   │  │  Cowork  │  │     Code      │ │
│  └──────────┘  └────┬─────┘  └───────────────┘ │
│                     │ SDK protocol (pipes)       │
│  ┌──────────────────┴────────────────────────┐  │
│  │  MCP Server (host process)                │  │
│  │  Option A: gws mcp -s drive,sheets        │  │
│  │  Option B: uvx workspace-mcp              │  │
│  │  Option C: uvx mcp-google-sheets@latest   │  │
│  └──────────────────┬────────────────────────┘  │
└─────────────────────┼───────────────────────────┘
                      │ HTTPS (Google APIs)
                      ▼
            Google Workspace APIs
```

**Step-by-step setup:**

1. **Install your chosen tool on the host.** For `gws`: `npm install -g @googleworkspace/cli`. For the community server: `pip install workspace-mcp` or use `uvx`.

2. **Authenticate on the host.** For `gws`: run `gws auth setup` then `gws auth login` in Terminal. For community servers: configure OAuth credentials per their README.

3. **Add the MCP server to Claude Desktop's config.** Edit `~/Library/Application Support/Claude/claude_desktop_config.json` and add the server definition under `mcpServers`. Restart Claude Desktop.

4. **Open Cowork tab and start a task.** The MCP tools appear automatically. Ask Cowork to "list my recent Google Drive files" or "create a new spreadsheet with Q1 budget data" — it will use the MCP tools transparently.

## When you actually would need TCP/IP

TCP/IP bridging becomes relevant in only two edge cases. First, if you want to run the MCP server on a **different machine** (e.g., a cloud server with service account credentials), you'd use `mcp-remote` as a stdio-to-HTTP bridge: `"args": ["npx", "mcp-remote", "https://your-server.com/mcp"]`. Second, if you're wrapping a CLI tool that **doesn't support MCP** into an HTTP API using Flask-Shell2HTTP or a custom Express server — but `gws` already supports MCP natively, making this unnecessary.

The `supergateway` package (`npx -y supergateway`) converts between any MCP transport combination (stdio ↔ SSE ↔ WebSocket ↔ Streamable HTTP), useful if you need to bridge transport mismatches. And macOS sandboxed apps can connect to `localhost:*` by default, so even a local HTTP wrapper would be reachable — but again, unnecessary here.

## Conclusion

The user's original framing — needing TCP/IP to escape Cowork's sandbox — reflects a real constraint (the VM blocks arbitrary network and CLI access) but misidentifies the solution. **MCP is Cowork's designed escape hatch.** Any MCP server configured in Claude Desktop runs on the host with full system access and is transparently available inside Cowork's VM. The `gws` CLI's built-in MCP mode or community servers like `taylorwilsdon/google_workspace_mcp` slot directly into this architecture. The practical recommendation: start with `xing5/mcp-google-sheets` if your focus is Sheets automation, or `taylorwilsdon/google_workspace_mcp` if you need broader Workspace coverage. Use `gws mcp` only if you need the full 50+ API surface and accept pre-v1.0 instability. No TCP/IP required.
