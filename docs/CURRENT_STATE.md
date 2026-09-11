# Discord Setup MCP — Current State

## 1. Purpose

This project is a **MCP (Model Context Protocol) server** that lets an AI assistant manage a Discord server through a bot user. It creates and edits channels, categories, roles, permissions, and settings; posts content such as messages, embeds, forum posts, and webhook posts with personas; configures server features like AutoMod rules, scheduled events, invites, the Community feature, onboarding, and welcome screens; and applies full **custom blueprints** (described inline or loaded from a YAML/JSON file) idempotently.

A bot cannot create a blank Discord server — `POST /guilds` returns error 20001. The workflow is: you create the empty server in Discord, invite the bot with Administrator permissions, and then use this MCP to fill everything out.

## 2. Version

**3.0.0** (REST-only rewrite).

## 3. Architecture at a glance

```
src/
├── index.ts                 # Entry point — creates an McpServer instance and registers every tool
├── client/
│   ├── rest.ts              # Singleton REST(client) authenticated with the bot token; no gateway, no intents
│   └── config.ts            # Config loader: env DISCORD_BOT_TOKEN or ~/.discord-mcp/config.json (zod-validated)
├── services/
│   ├── guild.ts             # Guild resolution, info, channel/role fetching, list_guilds cache
│   ├── permissions.ts       # Single source of truth for SCREAMING_SNAKE permission name ↔ bigint bitfield mapping
│   ├── content.ts           # Pure helpers: buildEmbed, buildLinkButtons (with zod input validation)
│   ├── blueprint.ts         # Blueprint engine — schema (zod), YAML/JSON loader, classifyByName idempotency diff, plan/apply/export
│   ├── features.ts          # Pure helpers for AutoMod rules, scheduled events, image→data URI conversion
│   └── templates.ts         # Preset template application over REST
├── tools/                   # One .ts file per MCP tool group (each exports a toolDefinition JSON-schema, a zod InputSchema, and an async handler)
│   ├── automod.ts           # configure_automod
│   ├── blueprint.ts         # apply_blueprint, plan_blueprint, export_server
│   ├── branding.ts          # set_server_branding
│   ├── channels.ts          # create_category, create_channel, edit_channel, delete_channel
│   ├── community.ts         # enable_community, configure_onboarding, set_welcome_screen
│   ├── content.ts           # send_message, post_embed, pin_message, create_forum_post, post_via_webhook, post_message_with_components
│   ├── events.ts            # create_scheduled_event
│   ├── guild.ts             # list_guilds, select_guild, get_guild_info
│   ├── invites.ts           # create_invite
│   ├── roles.ts             # create_role, edit_role, delete_role, reorder_roles
│   ├── settings.ts          # update_server_settings, set_verification_level, set_content_filter, set_default_notifications
│   └── templates.ts         # list_templates, preview_template, apply_template
├── templates/               # 4 pre-built server presets (gaming, community, business, study-group)
│   ├── types.ts             # TypeScript interfaces for TemplateRole, TemplateChannel, etc.
│   ├── gaming.ts
│   ├── community.ts
│   ├── business.ts
│   └── study-group.ts
├── utils/
│   └── errors.ts            # DiscordMCPError hierarchy (14 subclasses) + wrapDiscordError that maps Discord API error codes to specific error types
```

**How a tool call works:**
Every tool module exports three things: a JSON Schema `*ToolDefinition`, a zod `*InputSchema`, and an async `*Handler`. In `index.ts` they are registered through `registerAsyncTool` or `registerSyncTool`, which wrap the handler in zod validation and error handling. The handler calls `resolveGuildId()` to pick a target server, then uses `getRest().post()/patch()/get()/delete()` via `@discordjs/rest`.

**Key libraries:**
- **Runtime:** Node.js 18+ (ES modules)
- **HTTP/Discord API:** `@discordjs/rest` with `discord.js` for route builders and permission constants; bucket-aware rate limiting built in
- **MCP SDK:** `@modelcontextprotocol/sdk` v1.29, stdio transport only (the optional Hono HTTP transport is never loaded)
- **Validation:** zod for all tool inputs and blueprint schemas
- **Blueprint format:** YAML/JSON files parsed by the `yaml` package

**No gateway connection.** The server makes REST API calls exclusively. There are no privileged intents, no WebSocket listener, instant cold start.

## 4. Tool groups (34 tools total)

### Guild tools
| Tool | What it does |
|------|-------------|
| `list_guilds` | Lists every Discord server the bot is a member of (id + name) |
| `select_guild` | Sets an active guild for subsequent tool calls by id or name |
| `get_guild_info` | Returns detailed info: name, member count, features, roles, channels, settings |

### Channel tools
| Tool | What it does |
|------|-------------|
| `create_category` | Creates a category to hold related channels |
| `create_channel` | Creates a text, voice, announcement, stage, forum, or media channel with optional topic, nsfw, slowmode, bitrate, user limit, and permission overwrites |
| `edit_channel` | Patches an existing channel's properties |
| `delete_channel` | Deletes a channel by id or name |

### Role tools
| Tool | What it does |
|------|-------------|
| `create_role` | Creates a role with color, hoist, mentionable, and SCREAMING_SNAKE permission list |
| `edit_role` | Patches an existing role's properties |
| `delete_role` | Deletes a role by id or name |
| `reorder_roles` | Sets the order in the role hierarchy |

### Settings tools
| Tool | What it does |
|------|-------------|
| `update_server_settings` | Patches multiple guild-level settings at once (name, description, verification level, content filter, notification setting) |
| `set_verification_level` | Sets account age/verification requirements (`none`, `low`, `medium`, `high`, `very_high`) |
| `set_content_filter` | Controls explicit content filtering (`disabled`, `members_without_roles`, `all_members`) |
| `set_default_notifications` | Default message notification setting for new members |

### Content tools (the "content layer")
| Tool | What it does |
|------|-------------|
| `send_message` | Posts text to a channel |
| `post_embed` | Posts a rich embed with title, description, fields, color, footer, image, thumbnail |
| `pin_message` | Pins an existing message in a channel |
| `create_forum_post` | Creates a post inside a forum channel |
| `post_via_webhook` | Posts through a webhook (supports custom persona name + avatar) |
| `post_message_with_components` | Posts a message with link buttons (style 5); interactive component clicks are not supported because there is no gateway listener |

### Blueprint tools (the core feature)
| Tool | What it does |
|------|-------------|
| `apply_blueprint` | Applies an inline blueprint or YAML/JSON file idempotently: creates missing entities, patches changed ones, skips identical ones. Seeded messages are posted only when a channel is newly created so re-applying never duplicates content. |
| `plan_blueprint` | Dry-run version of apply_blueprint: returns what would be created/updated/skipped plus warnings (Community-gated channels, role/channel limits). Zero mutations. |
| `export_server` | Reads a live server's structure and writes it back as an inline blueprint object — useful for cloning or backup. |

### Server-feature tools
| Tool | What it does |
|------|-------------|
| `configure_automod` | Creates an AutoMod rule (keyword, spam, mention-spam, keyword-preset triggers) that blocks matching messages and optionally alerts a channel |
| `create_scheduled_event` | Creates a Discord scheduled event in the server |
| `create_invite` | Creates an instant invite link to a channel or server |
| `enable_community` | Enables the Community feature (prerequisite for announcement/stage channels, onboarding, welcome screen) — requires Administrator permission |
| `configure_onboarding` | Configures Discord member onboarding prompts (requires Community) |
| `set_welcome_screen` | Sets the Welcome Screen shown when new members join (requires Community) |
| `set_server_branding` | Uploads a server icon and optionally sets the server banner |

### Template tools
| Tool | What it does |
|------|-------------|
| `list_templates` | Returns all built-in templates with summary info (name, role count, category count, channel count) |
| `preview_template` | Detailed preview of a template's roles and channels before applying |
| `apply_template` | Applies one of the 4 built-in presets: **gaming**, **community**, **business**, or **study-group** |

## 5. Blueprint engine (idempotency model)

The blueprint engine is at `services/blueprint.ts`. It works in three modes:

- **Load:** accepts either an inline object (`blueprint` parameter) or a file path on disk (`file` parameter). Both YAML and JSON are parsed by the same `yaml.parse()` function.
- **Classify (diff):** the core idempotency key is entity **name** (case-insensitive). For each desired entity it finds the live match, compares fields, and classifies as "create", "update", or "skip". Roles use a detailed field comparison (permissions bitfield, color, hoist, mentionable). Channels compare topic, nsfw flag, and slowmode.
- **Apply:** processes roles first (highest position to lowest), then categories then channels within each category. Messages defined in `channels[].messages[]` are seeded only on the channel's initial creation. A 400ms throttle between API calls prevents rate-limit issues.

Blueprint fields:
```yaml
guild:                          # optional server-level settings
  name: string                  # new server display name
  description: string           # server description
  verificationLevel: "low"      # none | low | medium | high | very_high
  contentFilter: "all_members"  # disabled | members_without_roles | all_members
  defaultNotifications:         # all_messages | only_mentions

roles:                          # list of roles (position controls hierarchy order)
  - name: string
    color: "#5865F2"            # hex or integer
    hoist: true                 # show separately in member list
    mentionable: true           # can be @mentioned
    permissions: ["VIEW_CHANNEL", "SEND_MESSAGES"]

categories:                     # ordered list of categories and their channels
  - name: INFO
    overwrites:                 # optional channel permission overrides
      - role: Team
        allow: [VIEW_CHANNEL]
        deny: []
    channels:
      - name: welcome
        type: text              # text | voice | announcement | stage | forum | media
        topic: "Welcome here"
        nsfw: false
        slowmode: 10            # seconds
        messages:               # seeded only on first creation (idempotent)
          - content: "Hello!"
            pin: true           # auto-pin the message after posting

guild:                          # optional guild settings (see above)
```

## 6. Template presets

| ID | Name | Use case | Roles | Categories | Channels |
|----|------|----------|-------|------------|----------|
| `gaming` | Gaming Server | Gaming community with voice and text channels for different games, team organization, and announcements | Multiple roles (Admin, Mod, Game-specific) | Gaming, Chat, Voice, Info | ~20+ |
| `community` | Community Server | General-purpose community server with structured categories for chat, support, and events | Admin, Moderator, Member roles | General, Support, Events, Social | ~15+ |
| `business` | Business Server | Professional team communication server | Roles by department (Management, Staff, Client) | Operations, Communication, Resources | ~12+ |
| `study_group` | Study Group | Academic/research group with shared resources and discussion channels | Role by subject/level | Discussion, Resources, Scheduling | ~10+ |

## 7. Configuration names (no secret values)

| Config name | Source | Required? | Description |
|-------------|--------|-----------|-------------|
| `DISCORD_BOT_TOKEN` | Environment variable or config file | Yes | The bot's authentication token from Discord Developer Portal |
| `defaultGuildId` | Config only (optional) | No | Default server ID used when a tool call does not specify one |
| `rateLimit.maxRetries` | Config or env (`DISCORD_RATE_LIMIT_MAX_RETRIES`) | No, default 3 | How many times to retry after a rate limit hit |
| `rateLimit.retryDelay` | Config or env (`DISCORD_RATE_LIMIT_RETRY_DELAY`) | No, default 1000 ms | Base delay between retries |

Config file location: `~/.discord-mcp/config.json` (JSON format). File permissions should be `600`.

## 8. Error handling model

All errors are wrapped into a class hierarchy under `DiscordMCPError`:
- **BotNotReadyError** — bot connection issue (recoverable)
- **InsufficientPermissionsError** — missing Discord API permission, lists required perms
- **GuildNotFoundError** — server not found or bot lacks access
- **GuildNotSelectedError** — no target guild in the call chain (recoverable)
- **RateLimitError** — rate-limited by Discord API (recoverable, includes retry-after ms)
- **ConfigurationError** — missing/invalid config (fatal until fixed)
- **ChannelNotFoundError** / **RoleNotFoundError** — entity not found
- **ValidationError** — tool input failed zod validation
- **TimeoutError** — operation exceeded time limit (recoverable)
- **DiscordStateError** — Discord is in an unexpected state (recoverable)
- **TemplateError** — bad template id or name
- **CommunityRequiredError** — operation needs COMMUNITY feature enabled

`wrapDiscordError()` maps specific Discord API error codes to the most specific subclass (e.g., code 50013 → InsufficientPermissionsError, code 20001 → raw DiscordMCPError). Every tool handler catches its own exceptions and returns `{ success: false, error: ... }` rather than throwing.

## 9. Test suite

**Unit tests** (vitest, no network — `tests/*.test.ts`):
- `blueprint.test.ts` — loadBlueprint (inline, YAML file, JSON file), classifyByName idempotency diff, BlueprintZ validation edge cases
- `permissions.test.ts` — permission name → bitfield conversion, round-trip names, Nov-2025 split bits awareness, unknown-name safety
- `content.test.ts` — embed builder input validation and output structure
- `features.test.ts` — AutoMod rule body builders, scheduled event body builders
- `guild-resolve.test.ts` — guild resolution priority chain (explicit → current → config default), error cases

**Live checks** (against a dedicated test server only):
- `tests/live-smoke.mjs` — REST foundation smoke: list_guilds, select_guild, create/edit/delete channel and role, cleanup
- `tests/live-smoke-content.mjs` — content layer smoke: send_message, post_embed, pin_message
- `tests/_blueprint-livecheck.ts` — idempotency live check on blueprint apply/plan/export
- `tests/_features-livecheck.ts` — AutoMod rule creation and event scheduling live test
- `tests/_mcp-handshake.mjs` — verifies MCP stdio handshake (initialize + tools/list) works end-to-end

## 10. Known limits

| Limit | Detail |
|-------|--------|
| **Cannot create a blank server** | Discord rejects `POST /guilds` with code 20001 for bots. The server must already exist. |
| **Role hierarchy constraint** | The bot can only manage roles below its own position in the hierarchy and can grant only permissions it holds itself. |
| **Interactive components unsupported** | Button/select clicks require a persistent gateway listener. This server is request/response over stdio — link buttons (style 5) are supported; all other components are not. |
| **Community-gated features** | Announcement channels, stage channels, onboarding prompts, and welcome screens all require the guild to have the COMMUNITY feature enabled first. Run `enable_community` (requires Administrator). |
| **Vanity URL & Server Tags** | No bot-accessible write API for these. |
| **Banner / splash images** | Boost-gated features; only server icon upload is fully supported. |
| **Maximum roles** | 250 per guild. `plan_blueprint` warns if applying would exceed this. |
| **Maximum channels** | 500 per guild. `plan_blueprint` warns if applying would exceed this. |

## 11. Repository information

| Item | Value |
|------|-------|
| Remote origin | `https://github.com/gamween/discord-setup-mcp.git` (fetch + push) |
| Default branch | `main` |
| Current commit | `756f40b` — "docs: clearer step-by-step bot setup" |
| License | MIT |
| Install script | `install.sh` (one-liner curl command to clone and install) |

## 12. CI/CD

- GitHub Actions workflow in `.github/workflows/ci.yml` (runs lint/typecheck + tests on push/PR).
