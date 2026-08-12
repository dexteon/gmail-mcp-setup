# Multi-Account Gmail MCP Setup for Claude Code

How to wire **multiple Gmail accounts** into Claude Code (or any MCP client) as separate MCP servers, using the open-source [`@gongrzhe/server-gmail-autoauth-mcp`](https://github.com/GongRzhe/Gmail-MCP-Server) package — one server instance per account, all sharing a single Google Cloud OAuth client.

Each account shows up as its own MCP server (e.g. `gmail-personal`, `gmail-work`), so an agent can search, read, label, draft, and send from any of them, and tool names stay unambiguous (`mcp__gmail-personal__search_emails` vs `mcp__gmail-work__search_emails`).

## How it works

- The server is a stock npm package — you don't clone or build anything; `npx` fetches it on demand.
- One **GCP OAuth client** (a `gcp-oauth.keys.json` file) is shared by all account instances.
- Each account gets its **own credentials file** (the OAuth refresh token minted when that account approves access).
- Two env vars point each server instance at its files:
  - `GMAIL_OAUTH_PATH` — the shared OAuth client keys
  - `GMAIL_CREDENTIALS_PATH` — that account's token file

## 1. Create the OAuth client (once)

1. In [Google Cloud Console](https://console.cloud.google.com/), create a project and enable the **Gmail API**.
2. Configure the OAuth consent screen (External + test users is fine for personal use; add each Gmail address you'll connect as a test user).
3. Create an **OAuth client ID** of type **Desktop app**, download the JSON, and save it as:

```
~/.gmail-mcp/gcp-oauth.keys.json        # Windows: C:\Users\<you>\.gmail-mcp\gcp-oauth.keys.json
```

## 2. Authenticate each account (once per account)

Run the package's auth flow once per account, pointing at a distinct credentials path each time:

```powershell
# Account 1
$env:GMAIL_OAUTH_PATH = "C:\Users\<you>\.gmail-mcp\gcp-oauth.keys.json"
$env:GMAIL_CREDENTIALS_PATH = "C:\Users\<you>\.gmail-mcp\credentials-personal.json"
npx -y @gongrzhe/server-gmail-autoauth-mcp auth

# Account 2 — same OAuth keys, different credentials path, sign in as the other account
$env:GMAIL_CREDENTIALS_PATH = "C:\Users\<you>\.gmail-mcp\credentials-work.json"
npx -y @gongrzhe/server-gmail-autoauth-mcp auth
```

A browser window opens for Google sign-in each time; pick the matching account. Each run writes a token file to the `GMAIL_CREDENTIALS_PATH` you set.

## 3. Register one MCP server per account

In Claude Code, MCP servers live in `~/.claude.json` under `mcpServers` (or use `claude mcp add`). One entry per account:

```jsonc
{
  "mcpServers": {
    "gmail-personal": {
      "type": "stdio",
      "command": "cmd",                       // Windows; on macOS/Linux use "npx" directly as the command
      "args": ["/c", "npx", "-y", "@gongrzhe/server-gmail-autoauth-mcp"],
      "env": {
        "GMAIL_CREDENTIALS_PATH": "C:\\Users\\<you>\\.gmail-mcp\\credentials-personal.json",
        "GMAIL_OAUTH_PATH": "C:\\Users\\<you>\\.gmail-mcp\\gcp-oauth.keys.json"
      }
    },
    "gmail-work": {
      "type": "stdio",
      "command": "cmd",
      "args": ["/c", "npx", "-y", "@gongrzhe/server-gmail-autoauth-mcp"],
      "env": {
        "GMAIL_CREDENTIALS_PATH": "C:\\Users\\<you>\\.gmail-mcp\\credentials-work.json",
        "GMAIL_OAUTH_PATH": "C:\\Users\\<you>\\.gmail-mcp\\gcp-oauth.keys.json"
      }
    }
  }
}
```

Add as many entries as you have accounts — only the server name and `GMAIL_CREDENTIALS_PATH` change.

## 4. Tools you get (per account)

Each server exposes the full Gmail toolset, including:

- `search_emails`, `read_email` — Gmail query-syntax search and message fetch
- `draft_email`, `send_email` — compose (agents should default to **draft-only** unless explicitly told to send)
- `modify_email`, `batch_modify_emails` — archive, mark read, move labels in bulk
- `list_email_labels`, `create_label`, `update_label`, `delete_label`, `get_or_create_label`
- `create_filter`, `list_filters`, `get_filter`, `delete_filter`, `create_filter_from_template`
- `download_attachment`

## Gotchas (learned the hard way)

- **`invalid_grant` needs a full client restart, not a reconnect.** When a refresh token dies, re-running the `auth` step writes fresh credentials — but a running MCP server keeps the old token in memory, and an MCP "reconnect" only re-establishes the JSON-RPC channel to the same process. Restart the whole Claude Code session (or kill the specific `node.exe` running the server) so the new credentials file is actually re-read.
- **Never commit the `.gmail-mcp` directory.** `gcp-oauth.keys.json` is your OAuth client secret and each `credentials-*.json` is a live refresh token for that mailbox. Keep them out of every repo; transfer between machines via a password manager or encrypted channel, or just re-run the auth flow on the new machine (usually easier).
- **Google may expire test-user tokens.** If the consent screen stays in "Testing" mode, refresh tokens can expire after 7 days of inactivity. Publishing the app (even unverified for personal scopes) or periodic use avoids surprise re-auths.
- **Windows needs the `cmd /c npx` wrapper** shown above; calling `npx` directly as the command fails to spawn under some Claude Code versions.
