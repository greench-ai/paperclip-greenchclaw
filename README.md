# paperclip-greenchclaw

**Run Paperclip agents on GreenchClaw — no Claude, no OpenAI, no cloud account.**

Paperclip ships with AI connections hardcoded to four cloud providers
(`anthropic`, `openai`, `openrouter`, `xai`). Its `openclaw_gateway` adapter
exists but **cannot be bound to any AI connection**, so a self-hosted runtime
like GreenchClaw is unreachable: agent updates fail with
`422 Select an AI connection compatible with the new harness and model`.

This patch adds a **`greenchclaw` provider** whose capability binds the
`openclaw_gateway` adapter, plus a `gateway` auth method. After it, you create a
GreenchClaw connection in Paperclip with just a gateway token and hire agents
that run on GreenchClaw.

Verified end-to-end: a Paperclip heartbeat → `ws://<greenchclaw>:18420` →
GreenchClaw executes the agent → `run succeeded`, full issue checkout flow.

## What it changes

| File | Change |
|---|---|
| `packages/shared/src/ai-connections.ts` | add `greenchclaw` to `AI_PROVIDERS`; add `gateway` to `aiAuthMethodSchema`; add capabilities entry (`greenchclaw` → `openclaw_gateway`); accept gateway in create schema |
| `server/src/routes/ai-connections.ts` | allow `gateway` method; skip vendor API-key validation for `greenchclaw` |
| `server/src/routes/agents.ts` | skip the vendor "hello probe" for `greenchclaw` (no vendor CLI to probe) |
| `server/src/services/agent-ai-connection-default.ts` | add `greenchclaw` env keys |
| `server/src/services/local-ai-credentials.ts` | `greenchclaw` uses a gateway token, not a subscription login |

## Install (source build)

```bash
git clone https://github.com/paperclipai/paperclip.git
cd paperclip
git apply /path/to/greenchclaw-provider.patch
pnpm install
pnpm build            # needs Node >=24.11 and Rust (rustup) for the runner binary
```

Then create the connection (replace ids/token):

```bash
curl -X POST -H 'Content-Type: application/json' \
  -d '{"provider":"greenchclaw","method":"gateway","name":"GreenchClaw Gateway",
       "ownership":"shared","apiKey":"<greenchclaw-gateway-token>","allAgents":true}' \
  http://127.0.0.1:3100/api/companies/<companyId>/ai-connections
```

And point an agent at it:

```bash
paperclipai agent update <agentId> --payload-json '{
  "adapterType":"openclaw_gateway",
  "adapterConfig":{"url":"ws://127.0.0.1:18420","headers":{"x-openclaw-token":"<token>"}},
  "runtimeConfig":{"aiConnection":{"provider":"greenchclaw","method":"gateway",
     "mode":"shared","connectionId":"<connId>","grantId":"<grantId>"}}}'
```

Note: `runtimeConfig.aiConnection` is the correct field (not a top-level
`aiConnectionBinding`); a wrong field silently no-ops.

## Apply to an installed (npm) Paperclip

If you run the npm-managed install, patch the shipped files instead of rebuilding:

- `~/.paperclip/cli/installs/npm/<ver>/node_modules/@paperclipai/shared/dist/ai-connections.js`
- `~/.paperclip/cli/installs/npm/<ver>/node_modules/@paperclipai/server/dist/routes/{ai-connections,agents}.js`
- `~/.paperclip/cli/installs/npm/<ver>/node_modules/@paperclipai/server/dist/services/{agent-ai-connection-default,local-ai-credentials}.js`

Note the **4-space indentation** in compiled output (source is 2-space) —
a search/replace that assumes source style silently misses.

## Gotchas found the hard way

- The CLI bundle (`paperclipai/dist/index.js`) and the server both import
  `@paperclipai/shared` **at runtime from the installed package** — patching the
  bundle alone is not enough; the shared package's compiled JS is the authority.
- GreenchClaw's gateway token is 48 hex chars; grabbing it with the wrong tool
  can return a truncated value → `token_mismatch` at the WS handshake.
- Agents need `~/.openclaw/workspace/paperclip-claimed-api-key.json` (from
  `paperclipai token agent create`) to call back into Paperclip's API — without
  it, heartbeats connect but the task flow 401s.

---

## Onboarding integration (added 2026-09-25)

The patch also wires GreenchClaw into Paperclip's **agent onboarding**: choosing a
GreenchClaw/gateway agent offers a "Connect GreenchClaw" step that asks for the
gateway token — no Claude/OpenAI sign-in required.

UI changes:
- `ui/src/components/ai-connections/model.ts` — provider label + `Gateway token` method label
- `ui/src/components/ai-connections/AiConnectionAuth.tsx` — token card for the `gateway` method
- `ui/src/components/ai-connections/AiConnectionCredentialStep.tsx` — `greenchclaw` uses the token step, saves with `method: "gateway"`
- `ui/src/components/ai-connections/AiConnectionField.tsx` — `aiProviderForAdapter` maps `openclaw_gateway → greenchclaw` (this is what makes onboarding auto-select it)
- `ui/src/components/ai-connections/ManagedAiConnectionDetails.tsx` — accept the new provider/method in the row view
- `ui/src/components/new-agent/AgentProviderConnection.tsx` — widen method unions to include `gateway`

## What still needs upstream (honest scope)

This is a **fork patch**, not an upstream release. A clean upstream contribution
would need the provider list, capabilities table, and validation relaxed to be
data-driven rather than a 4-entry literal. As shipped here it's a working,
verified integration; the "pick any connector, nothing forced" wizard goal is
met for GreenchClaw specifically.

---

## Upstream PR

This patch was submitted upstream as **paperclipai/paperclip#14006** —
"feat(ai-connections): support self-hosted providers (Ollama, GreenchClaw)".
12 files, +119/-19. Also adds **Ollama** (local, no credential) alongside GreenchClaw.

Branch: `greench-ai:feat/self-hosted-ai-connections`.
