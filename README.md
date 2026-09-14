# ByMorning Gateway

ByMorning provides one access layer between AI clients, model providers, and internal tools, with OpenAI-, Anthropic-, Google-, and MCP-compatible endpoints plus a built-in Chat client.

> [!NOTE]
> ByMorning is focused on teams operating in constrained environments, where centralized control over model and tool access is essential.

This repository is the public, local Docker distribution. It runs as one container and persists its database and files in one Docker volume.

## Features

- **LLM Gateway** — connect multiple model Providers behind OpenAI Chat Completions, OpenAI Responses, Anthropic Messages, and Google GenerateContent APIs.
- **MCP Gateway** — connect ByMorning to remote MCP servers and expose allowed Workspace Tools to external MCP clients.
- **Model control** — enable Models, restrict Providers, set Model allowlists, and route requests to another Model.
- **Tool permissions** — allow, deny, or require approval for Tool actions and resources.
- **Budgets and limits** — set monthly spend limits plus Workspace-wide requests-per-minute and tokens-per-minute caps.
- **Usage** — inspect invocations, tokens, and known spend by Model and user.
- **Chat** — use the built-in client for Sessions, Tools, approvals, Projects, files, Skills, and subagents.

## Quick start

### Docker Compose

```sh
git clone https://github.com/bymorning/releases.git
cd releases
docker compose up -d --wait
```

Open <http://localhost:3210>. Use `localhost`, not `127.0.0.1`, so the browser origin matches the configured `APP_ORIGIN`.

### Docker

```sh
docker run --name bymorning -d \
  -p 127.0.0.1:3210:3210 \
  -e HOST=0.0.0.0 \
  -e PORT=3210 \
  -e APP_ORIGIN=http://localhost:3210 \
  -v bymorning-gateway-data:/data \
  ghcr.io/bymorning/gateway:latest
```

The image is public; pulling it does not require a GitHub login.

Check health:

```sh
curl http://localhost:3210/system/health
```

## How it fits

```text
Claude Code ─┐                         ┌─ OpenAI
Codex ───────┤   OpenAI / Anthropic   ├─ Anthropic
Your apps ───┼──────► ByMorning ──────┼─ Google
MCP clients ─┤         Gateway         ├─ Bedrock
Chat ────────┘            │            └─ other Providers
                          │
                          └─ remote MCP servers and Workspace Tools
```

All clients use the Workspace's effective Models, Provider admission, limits, routing, and usage accounting. Tool callers use the Workspace Tool catalog and Permission rules appropriate to their entrypoint.

## Configure the Gateway

Start the container, open <http://localhost:3210>, and continue with local sign-in. The default local identity and a default Workspace are created without an external identity provider.

You can configure ByMorning in two ways:

1. **Console** — use the web UI for Providers, Models, MCP integrations, permissions, budgets, service accounts, and usage.
2. **Ask ByMorning** — in Chat, ask it to inspect or update supported policy settings, connect MCP servers, authorize Integrations, or manage Skills. The Toolkit still enforces Permission rules and asks for approval when required.

### 1. Connect Models and Providers

Open **AI Gateway → Models → Providers**, then add a Provider and Connection. Stored credentials remain managed Connections; supported environment credentials are discovered automatically.

| Provider | Environment credentials |
| --- | --- |
| Bymorning | `BYMORNING_API_KEY` |
| OpenAI | `OPENAI_API_KEY` |
| Anthropic | `ANTHROPIC_API_KEY` |
| Google | `GOOGLE_API_KEY`, `GOOGLE_GENERATIVE_AI_API_KEY`, or `GEMINI_API_KEY` |
| OpenCode Zen | `OPENCODE_API_KEY` |
| Fireworks | `FIREWORKS_API_KEY` |
| Amazon Bedrock | `AWS_BEARER_TOKEN_BEDROCK` and `AWS_REGION`, or AWS access credentials and region |

After connecting a Provider, enable the Models the Workspace may use.

### 2. Call the LLM Gateway

Open **AI Gateway → Service accounts**, create a Service Account and key, and copy the `bm_…` key when shown. ByMorning stores only its hash.

Call the OpenAI-compatible endpoint:

```sh
curl http://localhost:3210/inference/openai/v1/chat/completions \
  -H "Authorization: Bearer bm_…" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "your-enabled-model",
    "messages": [{ "role": "user", "content": "Say hi" }]
  }'
```

Supported protocol surfaces:

| Protocol | Endpoint |
| --- | --- |
| OpenAI Chat Completions | `/inference/openai/v1/chat/completions` |
| OpenAI Responses | `/inference/openai/v1/responses` |
| Anthropic Messages | `/inference/anthropic/v1/messages` |
| Google GenerateContent | `/inference/google/v1beta/models/{model}:generateContent` |
| OpenAI model discovery | `/inference/openai/v1/models` |

Streaming is supported. If a selected destination cannot represent a request feature, ByMorning returns an explicit error rather than silently dropping it.

### 3. Connect MCP servers

Open **MCP Gateway → Add integration**, enter a name and a Streamable HTTP endpoint, then authenticate if required. The name becomes the Tool namespace; for example, server `linear` contributes actions such as `linear_create_issue`.

You can also ask Chat:

> Connect the Linear MCP server at `https://mcp.linear.app/mcp`.

ByMorning can save the server, start the OAuth flow, and load its Tool catalog through the Toolkit, subject to your Permission rules.

To use ByMorning from an external MCP client, copy the connection endpoint from **MCP Gateway** and add it as a Streamable HTTP MCP server. The client authorizes access to one Workspace through OAuth.

### 4. Set Tool permissions

Permission rules are ordered; the last matching rule wins. Effects are `allow`, `ask`, and `deny`.

```jsonc
{
  "permissions": [
    { "action": "*", "resource": "*", "effect": "ask" },
    { "action": "read", "resource": "*", "effect": "allow" },
    { "action": "read", "resource": "*.env", "effect": "deny" },
    { "action": "linear_search", "resource": "*", "effect": "allow" }
  ]
}
```

`ask` pauses an interactive Chat Session for approval. Inbound MCP cannot open an interactive approval, so both `ask` and `deny` fail closed there.

### 5. Restrict and route Models

Use Workspace configuration to admit Providers, restrict exact Models, cap activity, and remap requests:

```jsonc
{
  "providers": {
    "allow": ["openai", "anthropic"]
  },
  "gateway": {
    "limits": {
      "rpm": 300,
      "tpm": 500000
    },
    "models": ["gpt-5.6", "claude-sonnet-5"]
  },
  "routing": [
    {
      "from": { "provider": "openai", "id": "gpt-5.6" },
      "to": { "provider": "anthropic", "id": "claude-sonnet-5" }
    }
  ]
}
```

Or ask Chat:

> Allow only OpenAI and Anthropic. Cap the Workspace at 300 RPM and 500,000 TPM.

The Toolkit can read the current policy and update these supported fields without replacing unrelated Workspace configuration.

### 6. Set budgets and inspect usage

Open **AI Gateway → Budgets** to set monthly USD Spend Limits for the Workspace, members, or Service Accounts. A Holder without a limit remains unrestricted but still accrues spend.

Open **AI Gateway → Overview** to inspect:

- invocations;
- input, output, reasoning, cache-read, and cache-write tokens;
- known spend;
- usage by Model and user.

Gateway and Chat inference share the Workspace's RPM, TPM, and Workspace Spend Limit.

### 7. Use Chat

Chat is a client of the same Gateway configuration—not a separate model or tool environment. Start a Session to:

- work with an enabled Model;
- invoke built-in and MCP Tools;
- review Tool input, output, diffs, and errors;
- approve `ask` Permission requests;
- organize Sessions around Projects and files;
- use operational Toolkit Tools to configure policy, manage MCP servers, authorize Integrations, and manage Skills conversationally, subject to Permission rules and required approvals;
- save reusable Skills;
- delegate work to subagents.

## Local data

The `bymorning-gateway-data` volume contains everything needed by this local installation:

| Path | Contents |
| --- | --- |
| `/data/pglite` | PGlite database: Workspaces, configuration, usage, and application records |
| `/data/storage` | Filesystem-backed object storage |
| `/data/cookie.key` | Local login cookie key |

Back up the volume to retain the installation. Removing it drops Workspace data and signs users out. Existing Compose Postgres volumes are not imported.

To stop without deleting data:

```sh
docker compose down
```

To intentionally delete all local data:

```sh
docker compose down --volumes
```

## Update

```sh
docker compose pull
docker compose up -d --wait
```

The same volume is reused. Startup migrations run before the application begins serving requests.

## Change the port

If port `3210` is busy, update both the host port and `APP_ORIGIN`:

```yaml
ports:
  - "127.0.0.1:8080:3210"
environment:
  APP_ORIGIN: http://localhost:8080
```

Then open <http://localhost:8080>.

## Limits of this distribution

> [!NOTE]
> This local image runs as a single container and is not horizontally scaled. Contact ByMorning to discuss enterprise deployment and horizontal-scaling options.

- Designed for local, loopback access. Local Login accepts only loopback HTTP origins.
- One PGlite-backed application container.
- No bundled Code Interpreter runtime or Docker socket access.
- Filesystem storage only by default.
- Public HTTPS, production SSO, AWS, and GovCloud use separate deployment paths.

## License

The MIT license in this repository covers the install files only. The container image is not distributed under the MIT license.
