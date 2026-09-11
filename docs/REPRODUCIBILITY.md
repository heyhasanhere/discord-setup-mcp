# Discord Setup MCP — Reproducibility Guide

This document explains how to set up, verify, and restore this project from scratch. It assumes you have never run the code before and want a fully working installation on your machine.

## 1. Prerequisites

You need three things installed on your machine:

- **Node.js** version 18 or higher (the `node` command should report v18+)
- **npm** (comes bundled with Node.js)
- A **Discord account** and access to the [Discord Developer Portal](https://discord.com/developers/applications)

Verify Node.js is available:
```bash
node --version        # should show v18.x or higher
npm --version         # should report a version number
git --version         # needed only if cloning from the repository
```

## 2. Create a Discord Application and Bot

This section creates the bot that this MCP server controls with. Do it once — the same token works for every installation.

### Step A: Create the application

1. Open https://discord.com/developers/applications in your browser.
2. Click **New Application** near the top right.
3. Type a name (e.g., "Server Setup Bot") and click **Create**.
4. On the General Information page, copy the **Application ID** — you need it for the invite URL later.

### Step B: Create the bot user

1. In the left sidebar of your application page, click **Bot**.
2. If there is a prompt saying "Add Bot", click it and confirm.
3. Under the **Token** section, click **Reset Token**, confirm the warning dialog, and **copy the token string**. This string is secret — treat it like a password.

### Step C: Disable privileged intents (not needed)

The server is REST-only. Scroll down to **Privileged Gateway Intents** and leave all three toggles **off**:
- PRESENCE INTENT — off
- SERVER MEMBERS INTENT — off
- MESSAGE CONTENT INTENT — off

### Step D: Invite the bot to your first Discord server

A bot cannot create a blank server. You must invite it to one that already exists.

1. Open Discord and click the **+** button in the server sidebar → **Create My Own** → give it a name (e.g., "Test Server").
2. Build this URL, replacing `YOUR_APP_ID` with your Application ID from Step A:
   ```
   https://discord.com/oauth2/authorize?client_id=YOUR_APP_ID&scope=bot+applications.commands&permissions=8
   ```
3. Open the URL in a browser, select "Test Server" (or whatever you named it) in the dropdown, and click **Authorize**.
4. `permissions=8` means Administrator — this lets the bot manage roles, channels, permissions, webhooks, AutoMod rules, and the Community feature stack on your behalf.

## 3. Install the Project

Clone or download the repository, install dependencies, and build:

```bash
# Clone (if you have git)
git clone https://github.com/gamween/discord-setup-mcp.git
cd discord-setup-mcp

# Or use the one-liner installer from the repo root
chmod +x install.sh && ./install.sh
```

Install dependencies and build the TypeScript source into JavaScript:

```bash
npm install      # downloads node_modules (dependencies listed in package.json)
npm run build    # runs tsup, produces dist/index.js, dist/index.d.ts, dist/index.js.map
```

After `npm run build` succeeds you should see a `dist/` directory with at least these files:
- `dist/index.js` — the compiled server (executable Node.js file)
- `dist/index.d.ts` — TypeScript type declarations
- `dist/index.js.map` — source map for debugging

## 4. Configure Your Bot Token

Provide your bot token in one of two ways. Only do **one** — pick whichever fits your workflow.

### Option A: Environment variable (simplest)

Add this line to your shell configuration file (`~/.zshrc`, `~/.bash_profile`, etc.), then restart your terminal or run `source ~/.zshrc`:
```bash
export DISCORD_BOT_TOKEN="your-bot-token-here"
```

### Option B: Config file (recommended for Claude Desktop / Claude Code)

```bash
mkdir -p ~/.discord-mcp
cat > ~/.discord-mcp/config.json << 'EOF'
{ "discordToken": "your-bot-token-here", "defaultGuildId": "" }
EOF
chmod 600 ~/.discord-mcp/config.json
```

If you know the Discord server ID of your primary server, put it in `defaultGuildId` (get it by enabling Developer Mode in Discord, right-clicking the server icon, and choosing Copy Server ID). If you leave it empty, you will use `select_guild` to pick a server manually.

Optional config keys (both default if omitted):
```json
{
  "discordToken": "...",
  "defaultGuildId": "",
  "rateLimit": {
    "maxRetries": 3,
    "retryDelay": 1000
  }
}
```

The config can also be controlled by environment variables: `DISCORD_RATE_LIMIT_MAX_RETRIES` and `DISCORD_RATE_LIMIT_RETRY_DELAY`.

## 5. Verify the Installation

Run these commands in order to check everything works end-to-end. Each one should succeed before you move to the next.

### Check 1 — Build passes

```bash
npm run build      # exits with code 0, no errors printed
```

### Check 2 — Type checking passes (no TypeScript errors)

```bash
npm run typecheck   # runs `tsc --noEmit`; should exit cleanly
```

### Check 3 — Unit tests pass

These tests do **not** contact Discord's API. They test pure logic in memory:

```bash
npm test            # runs vitest; all unit tests should pass
```

Expected test files run (from `tests/`):
- `blueprint.test.ts` — blueprint loading, idempotency diff classification, schema validation
- `permissions.test.ts` — permission name ↔ bitfield mapping, round-trip correctness
- `content.test.ts` — embed builder input validation and output structure
- `features.test.ts` — AutoMod rule body builders, scheduled event body builders
- `guild-resolve.test.ts` — guild ID resolution logic

### Check 4 — MCP handshake (stdio protocol)

This tests that the server starts and responds to a basic MCP initialize + tools/list call:

```bash
node tests/_mcp-handshake.mjs
```

Expected output includes a response with `capabilities.tools` and a list of tool names. If this fails, your Node.js installation or the compiled code has an issue.

### Check 5 — Live smoke test (optional but recommended)

These tests call the real Discord API against a dedicated test server. They create channels and roles then delete them to clean up. **Do not run these against a production server.**

```bash
node tests/live-smoke.mjs          # REST foundation: guild list, channel/role CRUD
node tests/live-smoke-content.mjs  # content layer: send_message, post_embed, pin_message
npx tsup tests/_blueprint-livecheck.ts --format esm --out-dir .livecheck --target node18 --no-dts && node .livecheck/_blueprint-livecheck.js   # blueprint engine idempotency test
npx tsup tests/_features-livecheck.ts --format esm --out-dir .livecheck --target node18 --no-dts && node .livecheck/_features-livecheck.js    # AutoMod and scheduled events live check
```

If all five checks pass, the installation is fully verified.

## 6. Connect to Claude Code (optional)

To use this MCP server from Claude Code:

```bash
claude mcp add -s user discord-setup node /absolute/path/to/discord-setup-mcp/dist/index.js
```

Replace `/absolute/path/to/discord-setup-mcp` with the actual path on your machine. The bot token is read from your environment variable or config file — you do not need to pass it on the command line.

After adding it, ask Claude: "list my Discord servers" and it should call `list_guilds`.

## 7. Connect to Claude Desktop (optional)

Edit this configuration file:
- **macOS:** `~/Library/Application Support/Claude/claude_desktop_config.json`
- **Windows:** `%APPDATA%\Claude\claude_desktop_config.json`

Add or merge the following entry into the `"mcpServers"` object:
```json
{
  "mcpServers": {
    "discord-setup": {
      "command": "node",
      "args": ["/absolute/path/to/discord-setup-mcp/dist/index.js"],
      "env": {
        "DISCORD_BOT_TOKEN": "your-bot-token-here"
      }
    }
  }
}
```

Restart Claude Desktop after editing the file. You should then see Discord tools available in your chat sidebar.

## 8. Restore from a clean machine (step-by-step)

Here is the complete sequence to go from nothing to working on a brand-new machine:

1. Install Node.js 18+ from https://nodejs.org
2. Open terminal, clone and install:
   ```bash
   git clone https://github.com/gamween/discord-setup-mcp.git
   cd discord-setup-mcp
   npm install
   npm run build
   ```
3. Go to https://discord.com/developers/applications → New Application → copy your Application ID.
4. In the Bot tab, reset token and copy it.
5. Create a config file:
   ```bash
   mkdir -p ~/.discord-mcp
   cat > ~/.discord-mcp/config.json << 'EOF'
   { "discordToken": "YOUR_TOKEN_HERE", "defaultGuildId": "" }
   EOF
   chmod 600 ~/.discord-mcp/config.json
   ```
6. Invite the bot to a Discord server using the OAuth2 URL with `permissions=8`.
7. Run verification checks:
   ```bash
   npm run typecheck && npm test && node tests/_mcp-handshake.mjs
   ```

If all checks pass, you are ready to use the MCP server.

## 9. Restore from a backup or corrupted state

### If `node_modules` is missing or corrupted:
```bash
rm -rf node_modules package-lock.json
npm install
npm run build
```

### If `dist/` is empty after changes to source code:
```bash
npm run clean    # if available; otherwise rm -rf dist/
npm run build
```

### If you need the config file again but forgot your token:

You must regenerate it in the Discord Developer Portal. There is no way to recover a lost bot token — go to Bot → Reset Token, copy the new one, and update `~/.discord-mcp/config.json` or the environment variable. Regenerating invalidates the old token immediately.

### If tests fail after a pull or checkout:
```bash
rm -rf node_modules package-lock.json
npm install
npm run build
npm test
```

## 10. Repository structure quick reference

```
discord-setup-mcp/
├── .github/workflows/ci.yml     # GitHub Actions CI (lint + tests)
├── blueprints/counsel.yaml      # Example blueprint file
├── docs/BOT_SETUP.md            # Step-by-step bot creation guide
├── docs/CURRENT_STATE.md        # This project's current state and architecture
├── docs/REPRODUCIBILITY.md      # This file — setup and restore from scratch
│── dist/                        # Compiled output (gitignored)
├── install.sh                   # One-liner installer script
├── package.json                 # Dependencies, scripts, metadata
├── src/                         # TypeScript source code
│   ├── index.ts                 # Entry point — registers all tools
│   ├── client/rest.ts           # REST client singleton
│   ├── client/config.ts         # Config loader (zod-validated)
│   ├── services/guild.ts        # Guild resolution, info, cache
│   ├── services/permissions.ts  # Permission name ↔ bitfield mapping
│   ├── services/content.ts      # Embed and link button builders
│   ├── services/blueprint.ts    # Blueprint engine: load/diff/plan/apply/export
│   ├── services/features.ts     # AutoMod/event body builders, image→dataURI
│   ├── services/templates.ts    # Template application over REST
│   ├── tools/*.ts               # 16 tool handler modules (one per group)
│   ├── templates/{types,gaming,community,business,study-group}.ts
│   └── utils/errors.ts          # Error class hierarchy + Discord error mapper
├── tests/                       # Unit and live test files
│   ├── *.test.ts                # Vitest unit tests (no network)
│   ├── live-smoke.mjs           # REST foundation live smoke test
│   ├── live-smoke-content.mjs   # Content layer live smoke test
│   ├── _blueprint-livecheck.ts  # Blueprint idempotency live check
│   ├── _features-livecheck.ts   # AutoMod/events live check
│   └── _mcp-handshake.mjs       # MCP stdio handshake verification
├── tsconfig.json                # TypeScript compiler options (ES2022, ESM)
├── tsup.config.ts               # Build config: ES modules, node18 target
└── vitest.config.ts             # Vitest test runner config
```
