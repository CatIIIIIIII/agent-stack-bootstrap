---
name: ubuntu-agent-stack-bootstrap
description: Recreate this user's audited OpenClaw and OpenCode agent stack on Ubuntu. Use when installing, migrating, repairing, or documenting OpenClaw with local gateway, IkunCode and DeepSeek providers, Headroom context compression, AgentMemory long-term memory, CodeGraph, OpenCode, Oh My OpenAgent, optional SearXNG search, and workspace skills. Excludes WeCom channel/plugin setup.
---

# Ubuntu Agent Stack Bootstrap

## Scope

Use this runbook to recreate the user's current agent environment on a fresh Ubuntu host. It captures the audited macOS machine state as of 2026-06-19 and translates paths/services to Ubuntu.

Do not copy secrets into logs or committed files. Use environment variables or a local password manager for:

- `IKUNCODE_API_KEY`
- `DEEPSEEK_API_KEY`
- `OPENCLAW_GATEWAY_TOKEN`

Explicitly exclude WeCom for now: do not migrate `channels.wecom`, WeCom bindings, WeCom plugin config, bot IDs, or WeCom secrets.

## Audited Stack

- OpenClaw `2026.6.8`, local gateway on `127.0.0.1:18789`, token auth.
- Main OpenClaw model `ikuncode-gpt/gpt-5.5`; aliases: `gpt`, `opus`, `deepseek`.
- Subagents model `deepseek/deepseek-v4-pro`.
- Providers:
  - `ikuncode-gpt`: `https://api.ikuncode.cc/v1`, `openai-responses`, model `gpt-5.5`, context `258000`, output `4096`, text+image, reasoning.
  - `ikuncode-opus`: `https://api.ikuncode.cc/v1`, `openai-completions`, model `claude-opus-4-8`, context `1000000`, output `4096`, text+image, reasoning.
  - `deepseek`: `https://api.deepseek.com`, `openai-completions`, model `deepseek-v4-pro`, output `4096`, text, reasoning.
- OpenClaw tools profile `coding`; SearXNG search configured at `http://127.0.0.1:8080`.
- OpenClaw plugins:
  - `headroom` enabled, context engine slot, current plugin source `~/headroom/plugins/openclaw`, version `0.26.0`, upstream `https://github.com/chopratejas/headroom`, audited commit `9f7f3adf`.
  - `agentmemory` enabled, memory slot, server `http://localhost:3111`, plugin version `0.9.4`, global server package `@agentmemory/agentmemory@0.9.27`.
  - `workboard` enabled.
- OpenClaw skill workshop: autonomous enabled, `approvalPolicy: pending`, `maxPending: 50`, `maxSkillBytes: 40000`.
- Internal OpenClaw hooks enabled: `session-memory`, `compaction-notifier`, `command-logger`, `bootstrap-extra-files`, `boot-md`, `self-improvement`.
- OpenCode `1.17.8`, OMO plugin `oh-my-openagent@latest`, default agent `Sisyphus - ultraworker`, default model `ikuncode/claude-opus-4-8`, small model `deepseek/deepseek-v4-pro`.
- CodeGraph `@colbymchenry/codegraph@1.0.1`, MCP command `codegraph serve --mcp`.
- Other global tools: `@openai/codex@0.141.0`, `clawhub@0.22.0`, `mcporter@0.12.0`, Bun `1.3.14`, Node `v26.3.0`, npm `11.16.0`, ast-grep `0.43.0`.

## Install Prerequisites

Run on Ubuntu with a normal sudo-capable user:

```bash
sudo apt update
sudo apt install -y curl git build-essential jq unzip ca-certificates \
  python3 python3-venv python3-pip pipx tmux ripgrep ffmpeg sqlite3 lsof
pipx ensurepath
```

Install Node 24+ or 26.x and Bun. Example with `fnm`:

```bash
curl -fsSL https://fnm.vercel.app/install | bash
export PATH="$HOME/.local/share/fnm:$PATH"
eval "$(fnm env --shell bash)"
fnm install 26
fnm default 26

curl -fsSL https://bun.sh/install | bash
export PATH="$HOME/.bun/bin:$PATH"
```

Install global packages pinned to the audited machine:

```bash
npm install -g openclaw@2026.6.8 opencode-ai@1.17.8 \
  @agentmemory/agentmemory@0.9.27 @colbymchenry/codegraph@1.0.1 \
  @openai/codex@0.141.0 clawhub@0.22.0 mcporter@0.12.0 \
  @ast-grep/cli@0.43.0
```

Verify:

```bash
openclaw --version
opencode --version
agentmemory --version || true
codegraph --version
sg --version
node --version
npm --version
bun --version
```

## Configure OpenClaw

Initialize OpenClaw without channel setup:

```bash
mkdir -p "$HOME/.openclaw/workspace"
openclaw onboard --non-interactive --accept-risk --mode local \
  --workspace "$HOME/.openclaw/workspace" \
  --gateway-auth token --gateway-token-ref-env OPENCLAW_GATEWAY_TOKEN \
  --gateway-port 18789 --gateway-bind loopback --tailscale off \
  --node-manager npm --skip-channels --skip-daemon --skip-ui
```

Patch the audited model, tool, plugin, skill, and hook settings. This writes real API keys from the environment into the local OpenClaw config; do not print the generated JSON in logs.

```bash
test -n "$IKUNCODE_API_KEY"
test -n "$DEEPSEEK_API_KEY"
test -n "$OPENCLAW_GATEWAY_TOKEN"

node <<'NODE' | openclaw config patch --stdin
const home = process.env.HOME
const req = (name) => {
  const value = process.env[name]
  if (!value) throw new Error(`${name} is required`)
  return value
}

const patch = {
  agents: {
    defaults: {
      workspace: `${home}/.openclaw/workspace`,
      model: { primary: 'ikuncode-gpt/gpt-5.5' },
      models: {
        'ikuncode-gpt/gpt-5.5': { alias: 'gpt' },
        'ikuncode-opus/claude-opus-4-8': { alias: 'opus' },
        'deepseek/deepseek-v4-pro': { alias: 'deepseek' }
      },
      subagents: { model: 'deepseek/deepseek-v4-pro' }
    }
  },
  gateway: {
    mode: 'local',
    auth: { mode: 'token', token: req('OPENCLAW_GATEWAY_TOKEN') },
    port: 18789,
    bind: 'loopback',
    tailscale: { mode: 'off', resetOnExit: false },
    controlUi: { allowInsecureAuth: true },
    nodes: {
      denyCommands: [
        'camera.snap', 'camera.clip', 'screen.record',
        'contacts.add', 'calendar.add', 'reminders.add',
        'sms.send', 'sms.search'
      ]
    }
  },
  session: { dmScope: 'per-channel-peer' },
  tools: {
    profile: 'coding',
    web: { search: { provider: 'searxng', enabled: true } }
  },
  models: {
    mode: 'merge',
    providers: {
      'ikuncode-gpt': {
        baseUrl: 'https://api.ikuncode.cc/v1',
        api: 'openai-responses',
        apiKey: req('IKUNCODE_API_KEY'),
        models: [{
          id: 'gpt-5.5',
          name: 'gpt-5.5 (Custom Provider)',
          contextWindow: 258000,
          maxTokens: 4096,
          input: ['text', 'image'],
          cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
          reasoning: true
        }]
      },
      'ikuncode-opus': {
        baseUrl: 'https://api.ikuncode.cc/v1',
        api: 'openai-completions',
        auth: 'api-key',
        apiKey: req('IKUNCODE_API_KEY'),
        timeoutSeconds: 300,
        models: [{
          id: 'claude-opus-4-8',
          name: 'claude-opus-4-8 (IkunCode Opus)',
          contextWindow: 1000000,
          maxTokens: 4096,
          input: ['text', 'image'],
          cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
          reasoning: true
        }]
      },
      deepseek: {
        baseUrl: 'https://api.deepseek.com',
        api: 'openai-completions',
        auth: 'api-key',
        apiKey: req('DEEPSEEK_API_KEY'),
        timeoutSeconds: 120,
        models: [{
          id: 'deepseek-v4-pro',
          name: 'deepseek-v4-pro (AgentMemory DeepSeek)',
          maxTokens: 4096,
          input: ['text'],
          cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
          reasoning: true
        }]
      }
    }
  },
  plugins: {
    entries: {
      searxng: {
        enabled: true,
        config: { webSearch: { baseUrl: 'http://127.0.0.1:8080' } }
      },
      headroom: {
        enabled: true,
        config: { pythonPath: '/usr/bin/python3' }
      },
      agentmemory: {
        enabled: true,
        hooks: { allowConversationAccess: true },
        config: {
          base_url: 'http://localhost:3111',
          token_budget: 2000,
          min_confidence: 0.5,
          fallback_on_error: true,
          timeout_ms: 5000
        }
      },
      workboard: { enabled: true }
    },
    load: {
      paths: [
        `${home}/headroom/plugins/openclaw`,
        `${home}/.openclaw/extensions/agentmemory`
      ]
    },
    slots: { contextEngine: 'headroom', memory: 'agentmemory' }
  },
  skills: {
    install: { nodeManager: 'npm' },
    entries: {
      'coding-agent': { enabled: true },
      trello: { enabled: false }
    },
    workshop: {
      autonomous: { enabled: true },
      approvalPolicy: 'pending',
      maxPending: 50,
      maxSkillBytes: 40000
    }
  },
  hooks: {
    internal: {
      enabled: true,
      entries: {
        'session-memory': { enabled: true },
        'compaction-notifier': { enabled: true },
        'command-logger': { enabled: true },
        'bootstrap-extra-files': { enabled: true },
        'boot-md': { enabled: true },
        'self-improvement': { enabled: true }
      }
    }
  }
}

process.stdout.write(JSON.stringify(patch, null, 2))
NODE
```

Never add WeCom entries while applying this patch.

Install and start the gateway as a user/system service:

```bash
openclaw config validate
openclaw gateway install --port 18789 --auth token --token "$OPENCLAW_GATEWAY_TOKEN" --bind loopback --tailscale off
openclaw gateway start
openclaw gateway status
```

Dashboard should be `http://127.0.0.1:18789/`.

## Install Headroom

For the closest match to this machine, use the local source checkout and linked OpenClaw plugin:

```bash
git clone https://github.com/chopratejas/headroom "$HOME/headroom"
cd "$HOME/headroom"
git checkout 9f7f3adf || true

pipx install 'headroom-ai[proxy]' || python3 -m pip install --user 'headroom-ai[proxy]'

cd "$HOME/headroom/plugins/openclaw"
npm install
npm run build
openclaw plugins install --dangerously-force-unsafe-install --link .
```

Start a managed user service for the proxy:

```bash
HEADROOM_BIN="$(command -v headroom || true)"
if [ -z "$HEADROOM_BIN" ]; then
  HEADROOM_BIN="$(python3 - <<'PY'
import shutil
print(shutil.which("headroom") or "/usr/bin/python3 -m headroom.cli")
PY
)"
fi

mkdir -p "$HOME/.config/systemd/user"
cat > "$HOME/.config/systemd/user/headroom.service" <<EOF
[Unit]
Description=Headroom proxy

[Service]
ExecStart=$HEADROOM_BIN proxy --host 127.0.0.1 --port 8787
Restart=on-failure
RestartSec=5

[Install]
WantedBy=default.target
EOF

systemctl --user daemon-reload
systemctl --user enable --now headroom
```

If the plugin install path differs, update `plugins.load.paths` in OpenClaw to point at the installed plugin. On Ubuntu, prefer `/usr/bin/python3` or omit `pythonPath`; do not reuse macOS `/usr/local/bin/python3` blindly.

## Install AgentMemory

Configure the server with DeepSeek and local embeddings:

```bash
mkdir -p "$HOME/.agentmemory"
cat > "$HOME/.agentmemory/.env" <<EOF
OPENAI_API_KEY=$DEEPSEEK_API_KEY
OPENAI_BASE_URL=https://api.deepseek.com
OPENAI_MODEL=deepseek-v4-pro
OPENAI_REASONING_EFFORT=low
OPENAI_TIMEOUT_MS=120000
AGENTMEMORY_LLM_TIMEOUT_MS=120000
MAX_TOKENS=4096

EMBEDDING_PROVIDER=local

AGENTMEMORY_AUTO_COMPRESS=true
CONSOLIDATION_ENABLED=true
GRAPH_EXTRACTION_ENABLED=true
GRAPH_EXTRACTION_BATCH_SIZE=8
SUMMARIZE_CHUNK_CONCURRENCY=6
EOF
chmod 600 "$HOME/.agentmemory/.env"
```

Install the OpenClaw memory integration from the AgentMemory repo:

```bash
tmp="$(mktemp -d)"
git clone https://github.com/rohitg00/agentmemory "$tmp/agentmemory"
mkdir -p "$HOME/.openclaw/extensions"
rm -rf "$HOME/.openclaw/extensions/agentmemory"
cp -R "$tmp/agentmemory/integrations/openclaw" "$HOME/.openclaw/extensions/agentmemory"
```

Apply the local session-safety patch if upstream still lacks it. Check first:

```bash
rg 'resolveSessionId|shouldSkipSession' "$HOME/.openclaw/extensions/agentmemory/plugin.mjs" || true
```

If missing, patch `plugin.mjs` so the plugin:

- adds `asNonEmptyString(value)`, `resolveSessionId(event, ctx)`, and `shouldSkipSession(sessionId)`.
- resolves the session from `event.sessionKey`, `ctx.sessionKey`, `event.sessionId`, `ctx.sessionId`, `event.runId`, `ctx.runId`, then fallback `openclaw-${Date.now()}`.
- skips session IDs starting with `agent:main:explicit:model-run-`, `model-run-`, `probe-`, `openclaw-hook-smoke-`, or `direct-observe-smoke-`.
- changes the `agent_end` handler signature from `(event)` to `(event, ctx)`.
- uses the resolved session ID before posting `/agentmemory/observe`.

Start the server as a user service:

```bash
AGENTMEMORY_BIN="$(command -v agentmemory)"
mkdir -p "$HOME/.config/systemd/user"
cat > "$HOME/.config/systemd/user/agentmemory.service" <<EOF
[Unit]
Description=AgentMemory server

[Service]
EnvironmentFile=%h/.agentmemory/.env
ExecStart=$AGENTMEMORY_BIN --port 3111 --verbose
Restart=on-failure
RestartSec=5

[Install]
WantedBy=default.target
EOF

systemctl --user daemon-reload
systemctl --user enable --now agentmemory
```

Health check:

```bash
curl -fsS http://127.0.0.1:3111/agentmemory/health
```

## Configure CodeGraph

OpenCode uses CodeGraph through MCP. Ensure `~/.config/opencode/opencode.jsonc` contains:

```json
{
  "mcp": {
    "codegraph": {
      "type": "local",
      "command": ["codegraph", "serve", "--mcp"],
      "enabled": true
    }
  }
}
```

Append this block to `~/.config/opencode/AGENTS.md` if it is absent:

```markdown
<!-- CODEGRAPH_START -->
## CodeGraph

In repositories indexed by CodeGraph (a `.codegraph/` directory exists at the repo root), reach for it BEFORE grep/find or reading files when you need to understand or locate code:

- **MCP tools** (when available): `codegraph_explore` answers most code questions in one call - the relevant symbols' verbatim source plus the call paths between them. `codegraph_node` returns one symbol's source + callers, or reads a whole file with line numbers. If the tools are listed but deferred, load them by name via tool search.
- **Shell** (always works): `codegraph explore "<symbol names or question>"` and `codegraph node <symbol-or-file>` print the same output.

If there is no `.codegraph/` directory, skip CodeGraph entirely - indexing is the user's decision.
<!-- CODEGRAPH_END -->
```

## Configure OpenCode and OMO

Install OMO non-interactively:

```bash
mkdir -p "$HOME/.config/opencode"
cd "$HOME/.config/opencode"
npm install @opencode-ai/plugin@1.17.8

bunx oh-my-openagent install --no-tui --platform=opencode \
  --claude=no --openai=no --gemini=no --copilot=no \
  --opencode-zen=no --zai-coding-plan=no --kimi-for-coding=no \
  --opencode-go=no --bailian-coding-plan=no \
  --minimax-cn-coding-plan=no --minimax-coding-plan=no \
  --vercel-ai-gateway=no --skip-auth
```

Render the audited OpenCode config from environment variables:

```bash
node <<'NODE'
const fs = require('fs')
const os = require('os')
const path = require('path')
const home = os.homedir()
const dir = path.join(home, '.config', 'opencode')
fs.mkdirSync(dir, { recursive: true })
const req = (name) => {
  const value = process.env[name]
  if (!value) throw new Error(`${name} is required`)
  return value
}

const config = {
  plugin: ['oh-my-openagent@latest'],
  $schema: 'https://opencode.ai/config.json',
  provider: {
    ikuncode: {
      name: 'IkunCode',
      api: 'openai',
      options: { apiKey: req('IKUNCODE_API_KEY'), baseURL: 'https://api.ikuncode.cc/v1' },
      models: {
        'claude-opus-4-8': {
          id: 'claude-opus-4-8',
          name: 'Claude Opus 4.8',
          family: 'claude',
          reasoning: true,
          tool_call: true,
          limit: { context: 1000000, output: 64000 }
        },
        'claude-haiku-4-5-20251001': {
          id: 'claude-haiku-4-5-20251001',
          name: 'Claude Haiku 4.5',
          family: 'claude',
          reasoning: true,
          tool_call: true,
          limit: { context: 200000, output: 64000 }
        }
      }
    },
    deepseek: {
      name: 'DeepSeek',
      api: 'openai',
      options: { apiKey: req('DEEPSEEK_API_KEY'), baseURL: 'https://api.deepseek.com' },
      models: {
        'deepseek-v4-pro': {
          id: 'deepseek-v4-pro',
          name: 'DeepSeek V4 Pro',
          family: 'deepseek',
          reasoning: true,
          tool_call: true,
          limit: { context: 128000, output: 8192 }
        }
      }
    }
  },
  model: 'ikuncode/claude-opus-4-8',
  small_model: 'deepseek/deepseek-v4-pro',
  default_agent: 'Sisyphus - ultraworker',
  agent: {
    explore: { model: 'deepseek/deepseek-v4-pro', mode: 'subagent' },
    general: { model: 'deepseek/deepseek-v4-pro', mode: 'subagent' },
    delegate: {
      description: 'Delegates focused coding subtasks to DeepSeek.',
      model: 'deepseek/deepseek-v4-pro',
      mode: 'subagent'
    }
  },
  mcp: {
    codegraph: { type: 'local', command: ['codegraph', 'serve', '--mcp'], enabled: true }
  }
}

fs.writeFileSync(path.join(dir, 'opencode.jsonc'), JSON.stringify(config, null, 2) + '\n')
fs.writeFileSync(path.join(dir, 'tui.json'), JSON.stringify({ plugin: ['oh-my-openagent@latest'] }, null, 2) + '\n')
fs.writeFileSync(path.join(dir, 'package.json'), JSON.stringify({ dependencies: { '@opencode-ai/plugin': '1.17.8' } }, null, 2) + '\n')
NODE
```

Write the audited OMO agent/category mapping:

```bash
cat > "$HOME/.config/opencode/oh-my-openagent.json" <<'JSON'
{
  "$schema": "https://raw.githubusercontent.com/code-yeongyu/oh-my-openagent/dev/assets/oh-my-opencode.schema.json",
  "agents": {
    "sisyphus": { "model": "ikuncode/claude-opus-4-8" },
    "hephaestus": { "model": "ikuncode/claude-opus-4-8" },
    "oracle": { "model": "ikuncode/claude-opus-4-8" },
    "librarian": { "model": "ikuncode/deepseek" },
    "explore": { "model": "ikuncode/deepseek" },
    "multimodal-looker": { "model": "ikuncode/claude-opus-4-8" },
    "prometheus": { "model": "ikuncode/claude-opus-4-8" },
    "metis": { "model": "ikuncode/claude-opus-4-8" },
    "momus": { "model": "ikuncode/claude-opus-4-8" },
    "atlas": { "model": "ikuncode/claude-opus-4-8" },
    "sisyphus-junior": { "model": "ikuncode/claude-opus-4-8" }
  },
  "categories": {
    "visual-engineering": { "model": "ikuncode/claude-opus-4-8" },
    "ultrabrain": { "model": "ikuncode/claude-opus-4-8" },
    "deep": { "model": "ikuncode/claude-opus-4-8" },
    "artistry": { "model": "ikuncode/claude-opus-4-8" },
    "quick": { "model": "ikuncode/deepseek" },
    "unspecified-low": { "model": "ikuncode/deepseek" },
    "unspecified-high": { "model": "ikuncode/claude-opus-4-8" },
    "writing": { "model": "ikuncode/claude-opus-4-8" }
  }
}
JSON
```

If `bunx oh-my-openagent doctor --json` rejects `ikuncode/deepseek` on the target OpenCode/OMO version, change those OMO entries to `deepseek/deepseek-v4-pro`. Keep the audited value unless validation fails.

## Optional SearXNG

This machine has OpenClaw configured for SearXNG at `127.0.0.1:8080`, but the audited process list did not show SearXNG running. Either run it or disable web search in OpenClaw.

Docker example:

```bash
mkdir -p "$HOME/.openclaw/searxng"
docker run -d --name searxng --restart unless-stopped \
  -p 127.0.0.1:8080:8080 \
  -v "$HOME/.openclaw/searxng:/etc/searxng" \
  searxng/searxng:latest
```

Disable if not using it:

```bash
openclaw config patch --stdin <<'JSON'
{ "tools": { "web": { "search": { "enabled": false } } } }
JSON
```

## Migrate Workspace Skills

Current workspace skills to preserve:

- `agent-browser`
- `self-improving-agent`
- `nature-paper-to-patent`
- `nature-reviewer`
- `nature-paper2ppt`
- `nature-citation`
- `nature-data`
- `nature-response`
- `nature-writing`
- `nature-reader`
- `nature-academic-search`
- `nature-polishing`
- `nature-figure`
- `ubuntu-agent-stack-bootstrap`

Copy from the old machine when available:

```bash
mkdir -p "$HOME/.openclaw/workspace/skills"
rsync -a OLD_HOST:~/.openclaw/workspace/skills/ "$HOME/.openclaw/workspace/skills/" \
  --exclude 'wecom*'
```

If using `scp` or a tarball instead, preserve executable bits under each skill's `scripts/` directory. Do not migrate WeCom-specific skills/config unless the user explicitly asks later.

## Validate

Run all checks after installation:

```bash
openclaw config validate
openclaw plugins doctor
openclaw gateway status
openclaw skills list | rg 'ubuntu-agent-stack-bootstrap|agent-browser|self-improving-agent|nature-'
curl -fsS http://127.0.0.1:3111/agentmemory/health
curl -fsS http://127.0.0.1:8787/stats || true
bunx oh-my-openagent doctor --json
codegraph --version
sg --version
opencode --version
```

Expected live services:

- OpenClaw gateway listening on `127.0.0.1:18789`.
- AgentMemory server listening on `127.0.0.1:3111`.
- Headroom proxy listening on `127.0.0.1:8787`.
- SearXNG on `127.0.0.1:8080` only if enabled.

## Troubleshooting

- If OpenClaw cannot load `headroom`, verify `~/headroom/plugins/openclaw/dist/index.js` exists and `plugins.load.paths` points at `~/headroom/plugins/openclaw`.
- If Headroom is installed with `pipx` but the service cannot find it, use the absolute path from `command -v headroom` in `headroom.service`.
- If AgentMemory pollutes memory with probe/model-run sessions, reapply the session-safety patch to `plugin.mjs`.
- If AgentMemory returns connection refused, inspect `systemctl --user status agentmemory` and confirm `~/.agentmemory/.env` has `OPENAI_API_KEY` and `OPENAI_BASE_URL`.
- If OMO doctor warns about ast-grep, confirm `sg --version`; install `@ast-grep/cli@0.43.0` globally.
- If OMO doctor rejects `ikuncode/deepseek`, replace those OMO mapping entries with `deepseek/deepseek-v4-pro`.
- If web search fails and SearXNG is not needed, disable OpenClaw web search rather than leaving a dead endpoint configured.
