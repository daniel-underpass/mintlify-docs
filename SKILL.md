---
name: luxxon
description: On-demand live vision for AI agents. Open a session at a point on Earth, receive WebRTC video or single JPEG frames, settle per-second in USDC on Base. Pre-funded pool, zero per-session signatures.
license: Proprietary
compatibility: HTTP/JSON API. Reference clients ship as `@luxxon/sdk` (TypeScript) and `@luxxon/mcp` (Model Context Protocol server). One-time on-chain `deposit` funds a pool; sessions debit it automatically. Settlement is on Base Sepolia today, Base mainnet to follow.
metadata:
  author: luxxon
  version: "v1"
  apiBase: https://api.luxxon.dev/api/v1
  docs: https://docs.luxxon.dev
  console: https://console.luxxon.dev
  mcpInstall: "npx -y --package=@luxxon/mcp luxxon-mcp"
  smithery: "https://smithery.ai/servers/luxxon"
  glama: "https://glama.ai/mcp/servers/luxxon-dev/luxxon-sdk"
  chainId: 84532
  contract: "0xa748C178df47c2DA2227C247293943713eB65a26"
allowed-tools: "mcp__luxxon__request_live_view, mcp__luxxon__get_session, mcp__luxxon__get_frame, mcp__luxxon__get_stream_url, mcp__luxxon__end_session, mcp__luxxon__cancel_session, mcp__luxxon__get_pricing_quote, mcp__luxxon__get_coverage, mcp__luxxon__get_wallet, mcp__luxxon__get_settlement"
---

# Luxxon — agent skill

Luxxon is a verifiable physical-reality API for AI agents. An agent
that needs to *see* a place on Earth — to verify a claim, score a
property, watch a corridor, inspect equipment — opens a Luxxon
session at a lat/lng. A human operator (or future drone/PTZ
camera) on the ground attaches a publisher via WHIP. The agent
either subscribes via WHEP for full WebRTC or polls
`GET /sessions/:id/frame` for single JPEG observations. Every
chargeable second settles on-chain in USDC at a rate locked at
session-create time.

## When to use this skill

- The agent needs *current* visual evidence of a location, not a
  cached photo or generated image.
- Bounded duration acceptable (10s – several minutes per session).
- The agent's principal has USDC on Base + a wallet (EOA or smart
  contract) that has run `LuxxonSettlement.deposit(amount)` once
  to fund the per-session pool.
- Cost sensitivity matters — per-second billing, no flat fee, no
  minimum.

## When *not* to use this skill

- Need to control camera (PTZ, drone flight path) — v1 is "operator
  shows you what they see," not "agent drives the device."
- Strict regional / regulatory restrictions on third-party live
  video capture in the target area. Operator self-attestation is
  the v1 trust tier; see `concepts/trust-model`.
- Latency below ~500ms required. WHEP delivers sub-second but not
  real-time-control-loop tight.

## Capabilities

| Capability | Endpoint / Tool | Purpose |
|---|---|---|
| Quote a price | `GET /pricing/quote` | Lock a rate before requesting (optional). |
| Open a session | `POST /sessions` | Reserve dispatch at a lat/lng with a duration cap. API rejects with `INSUFFICIENT_CREDIT` if the consumer's pool can't cover `rate × maxDuration`. |
| Auto-dispatch | `POST /sessions/:id/dispatch` | Server picks the longest-idle ONLINE operator inside coverage. Session goes straight to ASSIGNED, no signature step. |
| Subscribe to video | `GET /sessions/:id/viewer-token` → WHEP | Sub-second WebRTC playback. |
| Single observation | `GET /sessions/:id/frame` | One JPEG snapshot, decoded server-side. Agent-friendly: no WebRTC stack required. |
| End + settle | `POST /sessions/:id/end` | Closes session; relayer submits `settleFromPool(...)` on-chain. |
| Inspect outcome | `GET /sessions/:id` / `GET /settlements/:sessionId` | State machine + on-chain tx hash. |

## Required inputs

| Input | Source | Example |
|---|---|---|
| API key | `console.luxxon.dev` → mint key (workspace-scoped) | `lxxn_test_xxxxxx_...` |
| Wallet for `from` | Consumer's EOA or smart wallet on Base | `0xD80f8e592Fc...` |
| Funded pool balance | `LuxxonSettlement.deposit(amount)` on-chain | `5_000_000` µUSDC (= 5 USDC) |
| Latitude / longitude | Where the agent wants eyes | `lat: 3.346, lng: -76.522` |
| Max duration (s) | Cap on chargeable seconds | `60` |

## Constraints

- **Environment gating.** All API keys are `TEST` today (Base Sepolia). Mainnet (`LIVE`) lights up when the platform announces mainnet readiness. See `concepts/api-keys`.
- **Coverage gating.** A session at a lat/lng without an ONLINE operator inside their declared coverage circle returns `503 NO_COVERAGE`. The request still feeds the public demand heatmap at `luxxon.dev`.
- **Pool funding required.** The first wallet interaction is a one-time `approve` + `deposit` on `LuxxonSettlement`. After that, sessions debit the pool with zero signatures. `LuxxonSettlement.withdraw(amount)` pulls the balance back at any time — even while the contract is paused.
- **Per-second pricing locked at create.** No surprise charges. The on-chain meter invariant `toAmount + feeAmount ≤ rate × seconds` prevents over-billing even by a fully-trusted relayer; the consumer caps total exposure by sizing the pool.

## Agent quickstart (MCP)

Drop `@luxxon/mcp` into the agent runtime's MCP config (Claude
Desktop, Cursor, Claude Code, or any MCP-compatible host):

```json
{
  "mcpServers": {
    "luxxon": {
      "command": "npx",
      "args": ["-y", "--package=@luxxon/mcp", "luxxon-mcp"],
      "env": { "LUXXON_API_KEY": "lxxn_test_..." }
    }
  }
}
```

Or one-click install via [Smithery](https://smithery.ai/servers/luxxon)
or [Glama](https://glama.ai/mcp/servers/luxxon-dev/luxxon-sdk).

Then in conversation: *"Look at lat 3.346, lng -76.522 for 30
seconds and tell me what you see."* The agent picks
`mcp__luxxon__get_frame`, the host returns a JPEG content block,
the LLM describes the frame. Settlement happens automatically at
`/end`; the agent doesn't sign a tx — the relayer submits
`settleFromPool(...)` and the consumer's pool absorbs the charge.

## Agent quickstart (TypeScript SDK)

```ts
import { Luxxon } from "@luxxon/sdk";
const lx = new Luxxon({ apiKey: process.env.LUXXON_API_KEY });

const session = await lx.sessions.create({ lat: 3.346, lng: -76.522, maxDurationSeconds: 30 });
await lx.sessions.dispatch(session.id);

// Wait for state LIVE (poll /sessions/:id, ~2-5s typical).
const frame = await lx.sessions.frame(session.id); // Uint8Array, image/jpeg
```

The agent never signs anything. The wallet behind the workspace
funded the pool ahead of time; the relayer settles every session
out of that pool on-chain at `/end`.

## Failure modes the agent should handle

- `503 NO_COVERAGE` — no operator near the requested point. Try
  another location, schedule a bounty (when bounty surface ships),
  or surface "unavailable here" to the user.
- `400 INSUFFICIENT_CREDIT` — the consumer wallet's pool balance
  can't cover `rate × maxDuration`. Tell the principal to top up
  via the console or by calling `LuxxonSettlement.deposit(amount)`.
- `409 INVALID_STATE` — caller acted on a session that already
  transitioned (race with operator). Re-fetch state, retry.
- `404 FRAME_NOT_AVAILABLE` — session LIVE but first keyframe
  hasn't decoded yet (typical: 3–5s after `/start`). Retry with
  brief backoff. Also returned indefinitely for iOS-Safari
  publishers until the H.264 decode path lands (browser-to-browser
  WHEP playback still works).

## Related documentation

See the per-concept pages in this repo:

- `quickstart` — full curl walkthrough
- `concepts/sessions` — state machine
- `concepts/pricing` — rate math
- `concepts/wallet` — pool funding + withdraw
- `concepts/settlement` — `settleFromPool` + on-chain flow
- `concepts/trust-model` — operator attestation tiers
- `concepts/mcp` — MCP server usage
- `concepts/sdks` — TypeScript SDK
- `concepts/console` — browser dashboard for humans-with-wallets
