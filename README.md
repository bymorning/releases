<p align="center">
  <img src="assets/wordmark.svg" alt="ByMorning" width="520">
</p>

# ByMorning Gateway

Run one local gateway for your team's AI models and tools. ByMorning gives AI clients a shared endpoint while administrators control which providers, models, and tools are available, how much they can be used, and where requests are routed.

It supports OpenAI, Anthropic, and Google request formats, MCP clients and servers, and includes a built-in Chat client.

This repository contains the public single-container Docker distribution. Application data and files persist in one Docker volume.

## Quick start

```sh
git clone https://github.com/bymorning/releases.git
cd releases
docker compose up -d --wait
```

Open <http://localhost:3210>.

The default local identity and Workspace are created automatically; no external identity provider is required.

## Connect a model

1. Open **AI Gateway → Models → Providers**.
2. Add a provider and its credentials.
3. Enable at least one model.

ByMorning supports **15 model providers**: OpenAI, Anthropic, Google, OpenCode Zen, Fireworks, Amazon Bedrock, Groq, Cerebras, DeepInfra, Together AI, Mistral, DeepSeek, Baseten, OpenRouter, and xAI. You can also define a custom provider with its own endpoint, request format, and authentication.

## Call the gateway

Open **AI Gateway → Service accounts**, create a Service Account, and copy its `bm_…` key when shown. ByMorning stores only the key's hash.

```sh
curl http://localhost:3210/inference/openai/v1/chat/completions \
  -H "Authorization: Bearer bm_…" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "your-enabled-model",
    "messages": [{ "role": "user", "content": "Say hi" }]
  }'
```

Use the API format your client already supports:

| Protocol | Endpoint |
| --- | --- |
| OpenAI Chat Completions | `/inference/openai/v1/chat/completions` |
| OpenAI Responses | `/inference/openai/v1/responses` |
| Anthropic Messages | `/inference/anthropic/v1/messages` |
| Google GenerateContent | `/inference/google/v1beta/models/{model}:generateContent` |
| OpenAI model discovery | `/inference/openai/v1/models` |

Streaming is supported. Features that cannot be represented by the selected destination return an explicit error rather than being silently dropped.

## Call the MCP gateway

Point any Streamable HTTP MCP client at:

```text
http://localhost:3210/mcp
```

ByMorning uses MCP OAuth, so there is no API key to create or paste into the client. Connect to the endpoint, complete the browser authorization, and approve access to a Workspace. The client can then discover and call the Tools that Workspace exposes to it.

Tool discovery and calls follow the Workspace's Permission rules. Interactive `ask` approvals are not available to inbound MCP clients, so `ask` and `deny` both fail closed.

## Reduce tool overhead with Code Mode

ByMorning uses a confined runtime sandbox for Code Mode: a tool-use pattern where a model writes code instead of requesting each operation separately. The code can discover available Tools, call several of them in one execution, run independent calls in parallel, and combine their results before returning them to the model.

For multi-step work, this can reduce model round trips and the tokens spent describing intermediate Tool calls and results. The runtime cannot import packages or access the network or filesystem directly; external actions still go through the Tools exposed by ByMorning and remain subject to their Permission rules.

## Turn scripts into Tools

Upload a Python or JavaScript file in Chat, then ask ByMorning to create a Tool plugin from it:

> Create a Tool plugin from `@report.py`. Give it a `source` input and return the generated report.

ByMorning writes the plugin into the current Project, defines the Tool's inputs and outputs, and connects it to the uploaded script. The script runs in the configured runtime sandbox rather than in the ByMorning server process. Once the plugin activates, the Tool is available to Chat and can be exposed through the MCP gateway under the Workspace's Permission rules.

## What you can control

- **Model access** — connect providers, enable models, restrict provider and model access, and route requests between models.
- **Usage** — inspect requests, tokens, and known spend by model and user.
- **Limits** — set monthly budgets and Workspace-wide request and token caps.
- **Tools** — connect remote MCP servers and expose approved Workspace Tools to MCP clients.
- **Permissions** — allow, deny, or require interactive approval for Tool actions and resources.

All clients use the Workspace's effective model configuration, provider restrictions, limits, routing, and usage accounting.

## Configure with Chat

You do not have to navigate the Console for every change. Ask ByMorning in Chat and its operational Toolkit can inspect or update Workspace configuration, subject to the same Permission rules and approval flow as other Tools.

The Toolkit exposes 15 configuration Tools:

| Area | Tools | Example prompts |
| --- | --- | --- |
| Policy | `toolkit.policy.get`, `toolkit.policy.update` | “Show me the current gateway policy.”<br>“Allow only OpenAI and Anthropic, with a limit of 300 RPM and 500,000 TPM.”<br>“Route requests for `gpt-5.6` to `claude-sonnet-5`.”<br>“Allow read actions, deny access to `.env` files, and ask before everything else.” |
| MCP servers | `toolkit.mcp.list`, `toolkit.mcp.put`, `toolkit.mcp.remove`, `toolkit.mcp.connect`, `toolkit.mcp.disconnect`, `toolkit.mcp.status` | “List the configured MCP servers.”<br>“Add the Linear MCP server at `https://mcp.linear.app/mcp` and connect it.”<br>“Show the status of Linear.”<br>“Disconnect Jira without deleting its configuration.”<br>“Remove the Jira MCP server.” |
| Integration OAuth | `toolkit.integration.connect`, `toolkit.integration.status` | “Authorize my Linear Integration.”<br>“Check whether the Linear authorization completed.” |
| Skills | `toolkit.skill.list`, `toolkit.skill.get`, `toolkit.skill.create`, `toolkit.skill.update`, `toolkit.skill.delete` | “List the available Skills.”<br>“Create a release-review Skill that checks readiness and rollback plans.”<br>“Update the release-review Skill to include migration checks.”<br>“Delete the release-review Skill.” |

Ask for the outcome you want; you do not need to mention Tool names. ByMorning selects the appropriate Toolkit operations, shows approval requests when required, and preserves unrelated configuration when applying supported policy changes.

## Connect external MCP servers

Open **MCP Gateway → Add integration**, enter a name and Streamable HTTP endpoint, then authenticate if required. The integration name becomes the Tool namespace.

## Run with Docker

You can run the image without Compose:

```sh
docker run --name bymorning -d \
  -p 127.0.0.1:3210:3210 \
  -e HOST=0.0.0.0 \
  -e PORT=3210 \
  -e APP_ORIGIN=http://localhost:3210 \
  -v bymorning-gateway-data:/data \
  ghcr.io/bymorning/gateway:latest
```

The image is public and does not require a GitHub login. Check its health with:

```sh
curl http://localhost:3210/system/health
```

### Data

The `bymorning-gateway-data` volume contains the PGlite database, stored files, and local login cookie key. Back up this volume to retain the installation.

## Limitations

This image is intended for local, loopback use:

- one PGlite-backed application container;
- no horizontal scaling.

## License

The MIT license in this repository covers the install files only. The container image is not distributed under the MIT license.
