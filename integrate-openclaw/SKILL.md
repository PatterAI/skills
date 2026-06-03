---
name: integrate-openclaw
description: >
  Wire Patter as the voice layer on top of an OpenClaw brain — the
  "brain on the line" pattern. Use when the user runs OpenClaw (or any
  OpenAI-compatible agent gateway) and wants a phone number that answers
  any caller, talks with low latency, and consults a specific OpenClaw
  receptionist agent for real data/actions mid-call without the call
  dropping during a slow (30-60 s) tool. Covers both directions
  (OpenClaw drives Patter via `patter-mcp`; Patter consults OpenClaw via
  `ConsultConfig`), long-tool-call survival, speakerphone noise tuning,
  open inbound, and least-privilege agent scoping. Patter 0.6.3, Python
  and TypeScript.
license: MIT
compatibility: >
  Requires Patter >= 0.6.3 (`ConsultConfig` / `consult` shipped in 0.6.x)
  and a running OpenClaw gateway you control. Consult is injected in
  Realtime and Pipeline modes only (ElevenLabs ConvAI hosts its own tools,
  so consult does not apply there). For Direction A you also need the
  `patter-mcp` server from a local clone (it is NOT published to npm).
metadata:
  author: patter
  version: "0.1.0"
  parity: both
---

# Integrate Patter with OpenClaw (brain on the line)

[OpenClaw](https://openclaw.ai) is a self-hosted gateway of LLM-backed agents
with scoped tool connections (calendars, customer DBs, …). Patter is the **voice
layer**: it owns the carrier leg, talks with low latency, and reaches into
OpenClaw only when a turn needs real data or an action. This is the
**brain-on-the-line** architecture — the inverse of wiring OpenClaw in as a
"Custom LLM" that sits on every turn (the topology that kills calls on slow
tools).

```
CALLER (any number, no allowlist)
   │
   ▼
Twilio / Telnyx DID  ──►  PATTER voice agent (OpenAI Realtime, low-latency EN)
                              │  owns turn-taking + TTS locally
                              │  speaks "let me check, one moment"
                              ▼
                          consult ──► local adapter ──► OpenClaw
                              ▲          (loopback)        /v1/chat/completions
                              │                            model="openclaw/<receptionistAgentId>"
                          speaks the reply                 (ONE scoped agent)
```

## The two directions — and which one to use

| Direction | Who initiates | Transport | Use when |
|---|---|---|---|
| **A — OpenClaw drives Patter** | OpenClaw agent | `patter-mcp` MCP server (tools) | OpenClaw should **place outbound calls** ("call the clinic, wait, tell me what they said"). Blocking until the call ends is the *desired* semantics here. |
| **B — Patter consults OpenClaw** | In-call Patter agent | `ConsultConfig` → HTTP adapter → OpenClaw gateway | Patter answers an **inbound** call and needs the brain mid-conversation. **This is the brain-on-the-line pattern** the after-hours-receptionist use case needs. |

They compose, but for an inbound receptionist line **Direction B is the one to
build first**. The rest of this skill is mostly Direction B; Direction A is at
the end.

### Why not OpenClaw's native voice plugin?

OpenClaw's built-in voice plugin only accepts inbound calls from an
**allowlist** (`inboundPolicy: "allowlist"` + `allowFrom: [...]`); there is no
"accept anyone" mode. That is useless for a public after-hours line where random
people call. **Patter owns the DID and answers everyone**, then reaches the
receptionist over OpenClaw's *operator-side* `/v1/chat/completions` gateway — the
caller is never an OpenClaw "sender" subject to the allowlist. This is the single
clearest reason to put Patter in front.

---

## Direction B — Patter consults OpenClaw mid-call

### Step 1 — Enable OpenClaw's chat-completions gateway

OpenClaw exposes an OpenAI-compatible `POST /v1/chat/completions`. It is
**disabled by default**. Enable it in `~/.openclaw/openclaw.json` (JSON5):

```json5
{
  gateway: {
    http: {
      endpoints: {
        chatCompletions: {
          enabled: true,
        },
      },
    },
  },
}
```

<Warning>
**The gateway credential is operator-grade — the `model` field is NOT a security
boundary.** Selecting an agent in the `model` field picks the *persona*, but with
shared-secret (token/password) auth the gateway credential carries full operator
scope across the whole gateway. Real isolation comes from three things, none of
which is the token:

1. A **dedicated least-privileged receptionist agent** with a tight per-agent
   `tools.allow` / `tools.deny` (see Step 5).
2. **One gateway per client** — bind the gateway to **loopback / tailnet only**
   and never expose `/v1/chat/completions` to the public internet. On a single
   always-on Mac Mini per client (one OS user = one gateway = one credential
   set), this isolation is free.
3. An **auth credential on the gateway** (mirror OpenClaw's own gateway-auth
   guidance — do not stand up an unauthenticated, network-reachable endpoint).

Confirm the exact gateway bind/auth keys and the `model`-routing alias forms
against the live OpenClaw docs ([docs.openclaw.ai](https://docs.openclaw.ai))
before you ship — they are OpenClaw-side config, not Patter's.
</Warning>

### Step 2 — Route the consult to ONE specific receptionist agent

On `/v1/chat/completions` the `model` field is an **agent target**:

- `"openclaw"` / `"openclaw/default"` → the **default agent** — which may be the
  master / CEO / financial agent and can drift between environments. **Never use
  this from the voice layer.**
- `"openclaw/<receptionistAgentId>"` → one explicit, named agent. **Always pin
  this.** (Alias forms `openclaw:<id>` / `agent:<id>` are commonly accepted —
  verify against OpenClaw docs.)

Per client, set the agentId explicitly:

| Client | Receptionist agentId |
|---|---|
| Roofing contractor (CA) | `openclaw/roofing-ca-receptionist` |
| Home-automation contractor (FL) | `openclaw/home-fl-receptionist` |

### Step 3 — Point consult at the OpenClaw agent

**Recommended: the native target — no adapter.** Patter's `consult` speaks OpenClaw's
`/chat/completions` endpoint directly; point it at one scoped agent in a single line:

```python
from getpatter import ConsultConfig, OpenAIRealtime2

agent = phone.agent(
    engine=OpenAIRealtime2(),
    system_prompt="You are the after-hours receptionist...",
    consult=ConsultConfig.openclaw("receptionist"),
)
```

```typescript
import { openclawConsult, OpenAIRealtime2 } from "getpatter";

const agent = phone.agent({
  engine: new OpenAIRealtime2(),
  systemPrompt: "You are the after-hours receptionist...",
  consult: openclawConsult("receptionist"),
});
```

It targets `model="openclaw/receptionist"`, sends the call id as the OpenAI `user` field
plus the `x-openclaw-session-key` header (one OpenClaw session per call), reads the
operator-grade bearer from `OPENCLAW_API_KEY` (never logged), auto-enables `allow_loopback`
for the co-located gateway, and attaches a default "let me check" reassurance filler.
**Notify OpenClaw at call end** with `openclaw_post_call_notifier("receptionist")` /
`openclawPostCallNotifier("receptionist")` on `serve(on_call_end=...)` to post the call
record (caller, line, duration, transcript) to the same agent and session.

**Escape hatch — a hand-written adapter**, only when you need a custom request/response
mapping (e.g. a non-OpenClaw back office). Patter POSTs `{ request, call_id, caller,
callee }`; the adapter must:

1. Target **one named agent** (`model="openclaw/<receptionistAgentId>"`).
2. Carry `call_id` as the OpenClaw **session key** so a multi-turn call maps to
   **one** OpenClaw session (continuity), and forward `caller` for caller-ID
   prefetch of the customer record.
3. Return a **concise, spoken-ready** string (don't dump a raw API blob back
   into the voice context).

```python
# adapter.py — forwards Patter consult requests to ONE OpenClaw receptionist agent
import os
import httpx
from fastapi import FastAPI, Request

app = FastAPI()

OPENCLAW_URL = os.environ["OPENCLAW_URL"]          # e.g. http://127.0.0.1:18789/v1/chat/completions
OPENCLAW_TOKEN = os.environ["OPENCLAW_GATEWAY_TOKEN"]
RECEPTIONIST_AGENT = os.environ.get("OPENCLAW_AGENT", "openclaw/receptionist")

@app.post("/consult")
async def consult(req: Request):
    body = await req.json()
    request_text = body.get("request", "")
    call_id = body.get("call_id", "")
    caller = body.get("caller", "")

    payload = {
        "model": RECEPTIONIST_AGENT,                # ONE scoped agent, never "openclaw"
        "user": call_id,                            # one OpenClaw session per call
        "messages": [{
            "role": "user",
            # caller-id is forwarded so the brain can prefetch the customer record
            "content": f"[caller={caller}] {request_text}",
        }],
    }
    headers = {"Authorization": f"Bearer {OPENCLAW_TOKEN}"}

    # 75 s is sized to OpenClaw's worst-case tool time (see "timeout ladder").
    async with httpx.AsyncClient(timeout=75.0) as client:
        resp = await client.post(OPENCLAW_URL, json=payload, headers=headers)
        resp.raise_for_status()
        content = resp.json()["choices"][0]["message"]["content"]

    # Trim to a short, spoken-ready reply before it re-enters the voice context.
    return {"reply": content.strip()[:600]}
```

```typescript
// adapter.ts — forwards Patter consult requests to ONE OpenClaw receptionist agent
import express from "express";

const app = express();
app.use(express.json());

const OPENCLAW_URL = process.env.OPENCLAW_URL!;            // http://127.0.0.1:18789/v1/chat/completions
const OPENCLAW_TOKEN = process.env.OPENCLAW_GATEWAY_TOKEN!;
const RECEPTIONIST_AGENT = process.env.OPENCLAW_AGENT ?? "openclaw/receptionist";

app.post("/consult", async (req, res) => {
  const { request = "", call_id = "", caller = "" } = req.body ?? {};

  const payload = {
    model: RECEPTIONIST_AGENT,                              // ONE scoped agent, never "openclaw"
    user: call_id,                                          // one OpenClaw session per call
    messages: [{ role: "user", content: `[caller=${caller}] ${request}` }],
  };

  const resp = await fetch(OPENCLAW_URL, {
    method: "POST",
    headers: { "Content-Type": "application/json", Authorization: `Bearer ${OPENCLAW_TOKEN}` },
    body: JSON.stringify(payload),
    signal: AbortSignal.timeout(75_000),                    // see "timeout ladder"
  });
  const data = await resp.json();
  const content: string = data.choices[0].message.content;

  res.json({ reply: content.trim().slice(0, 600) });        // short, spoken-ready
});

app.listen(8000);
```

> The adapter is also where you can drop slow work onto an OpenClaw sub-agent
> (e.g. spawn a worker, return a short "checking…" and let a later turn fetch the
> result). Optional — the in-band reassurance below already covers dead air for
> a single 60 s lookup.

### Step 4 — Point the Patter agent at the adapter

`ConsultConfig` (Python) / `consult` (TS) auto-injects a `consult_agent` tool the
in-call agent can call. On a single Mac Mini the adapter is on loopback, so opt
in with `allow_loopback` / `allowLoopback`.

```python
from getpatter import Patter, Twilio, OpenAIRealtime2, ConsultConfig

phone = Patter(carrier=Twilio(), phone_number="+15550001234")

agent = phone.agent(
    engine=OpenAIRealtime2(),                    # low-latency English, strongest tool flow
    system_prompt=(
        "You are the after-hours receptionist for Acme Roofing. "
        "When the caller asks about appointments, availability, or account "
        "details, call `consult_agent` with a clear request. "
        "Say a brief 'let me check' first, then read back the answer."
    ),
    first_message="Thanks for calling Acme Roofing, how can I help?",
    consult=ConsultConfig(
        url="http://127.0.0.1:8000/consult",
        timeout_s=75.0,                          # sized to the brain's worst-case tool time
        allow_loopback=True,                     # adapter is co-located on this box
        headers={"Authorization": "Bearer <adapter-token>"},  # optional, never logged
        # tool_name / description default to "consult_agent" + a sensible prompt;
        # override description to steer WHEN the agent escalates.
    ),
)

phone.serve(agent)
```

```typescript
import { Patter, Twilio, OpenAIRealtime2 } from "getpatter";

const phone = new Patter({ carrier: new Twilio(), phoneNumber: "+15550001234" });

const agent = phone.agent({
  engine: new OpenAIRealtime2(),                 // low-latency English, strongest tool flow
  systemPrompt:
    "You are the after-hours receptionist for Acme Roofing. " +
    "When the caller asks about appointments, availability, or account " +
    "details, call consult_agent with a clear request. " +
    "Say a brief 'let me check' first, then read back the answer.",
  firstMessage: "Thanks for calling Acme Roofing, how can I help?",
  consult: {
    url: "http://127.0.0.1:8000/consult",
    timeoutMs: 75_000,                           // sized to the brain's worst-case tool time
    allowLoopback: true,                         // adapter is co-located on this box
    headers: { Authorization: "Bearer <adapter-token>" }, // optional, never logged
  },
});

await phone.serve({ agent });
```

**Defaults to know:** consult timeout defaults to `30 s` (`timeout_s=30.0` /
`timeoutMs: 30000`), the tool is named `consult_agent`, and consult is injected
in **Realtime and Pipeline** only (a warning is emitted if you set it with
ElevenLabs ConvAI).

---

## Long-tool-call survival (the literal blocker)

The prior failure with another platform was: the agent makes a 30-60 s tool call
(sometimes browser automation) and **the call dies**. The fix is three layers,
all of which must hold.

### 1. The timeout ladder — the call dies at the SHORTEST link

| Layer | Where | Set to | Why |
|---|---|---|---|
| Patter consult timeout | `ConsultConfig.timeout_s` / `consult.timeoutMs` | **75-90 s** | Above the brain's worst-case tool time. Default 30 s is too short for 30-60 s automation. |
| Adapter HTTP timeout | `httpx.AsyncClient(timeout=…)` / `AbortSignal.timeout(…)` | **= consult timeout** | The adapter must outlast the consult, not undercut it. |
| OpenClaw gateway `chatCompletions` timeout | OpenClaw config | **raised** | The gateway must not give up before the agent's tool returns. |
| Direction A: `mcp.servers.patter.timeout` | OpenClaw config | **600 s** | A blocking `make_call` runs for the whole call. |

> **Verify the per-server timeout is honoured.** OpenClaw's docs don't publish a
> default for the per-server MCP timeout, and an MCP client can impose its own
> read-timeout ceiling that silently caps a long blocking call regardless of config.
> After wiring, probe the server with `openclaw mcp doctor patter --probe` (a live
> connection check). `openclaw mcp status --verbose` shows the resolved timeout but
> does NOT open a connection.

### 2. Reassurance — speak a filler the instant the tool starts

Without a filler the caller hears dead air for the whole lookup and assumes the
line froze. Attach `reassurance` to the **consult tool** (or any slow tool) so
the agent speaks immediately and keeps the media stream alive while the brain
works.

The `consult_agent` tool is auto-built, so to give it reassurance the cleanest
path today is to add your **own** slow tool that wraps the consult, OR steer the
agent via the system prompt to say "let me check, one moment" before it calls
`consult_agent`. For a hand-built slow tool, set `reassurance` on the `Tool`
(Python) / `ToolDefinition` (TS):

```python
from getpatter import Tool

# reassurance is a field on the Tool dataclass (string shorthand → after_ms=1500,
# or a dict {"message": str, "after_ms": int}). Realtime mode only today.
check_schedule = Tool(
    name="check_schedule",
    description="Look up the caller's appointment in the calendar.",
    parameters={"type": "object", "properties": {"day": {"type": "string"}}, "required": ["day"]},
    handler=my_slow_handler,                         # may take 30-60 s
    reassurance="Let me check the calendar for you, one moment.",
)
```

```typescript
// In TS, reassurance and a per-tool timeoutMs are fields on the raw
// ToolDefinition (defineTool does not expose them — build the object directly).
const checkSchedule = {
  name: "check_schedule",
  description: "Look up the caller's appointment in the calendar.",
  parameters: {
    type: "object",
    properties: { day: { type: "string" } },
    required: ["day"],
  },
  handler: mySlowHandler,                            // may take 30-60 s
  reassurance: "Let me check the calendar for you, one moment.",
  timeoutMs: 60_000,                                 // raise off the 10 s default
};
```

<Warning>
**Reassurance is Realtime-only in 0.6.3.** In Pipeline mode the field is silently
ignored (the LLM has to generate its own filler); ConvAI doesn't expose it. For
the receptionist line, **use Realtime** — it is also the lowest-latency English
path and has the strongest tool flow.
</Warning>

### 3. Per-tool timeout

- **TypeScript:** `ToolDefinition.timeoutMs` exists (default 10 000 ms, clamped to
  300 000 ms). Raise it to `60_000` for slow tools — shown above. A timeout
  returns `{ error, fallback: true }` and is not retried.
- **Python:** the `@tool` / `Tool` surface has **no** per-tool timeout field yet —
  the generic webhook path uses a fixed 10 s. The way to get a long, configurable
  timeout in Python today is **the consult path** (`ConsultConfig.timeout_s`),
  which the handler honours directly. So for Python, route slow work through
  `consult` rather than a webhook tool until the per-tool timeout lands.

### Why this beats "OpenClaw as a Custom LLM"

The failure mode to avoid is wiring OpenClaw in as the **per-turn LLM** (a
"Custom LLM" / `/v1/chat/completions` on the critical path of every utterance).
Then the brain's full reasoning + tool latency blocks **every** response, and a
60 s tool stalls the turn with no acknowledge-then-continue primitive. Patter's
`consult` is the inverse: the **local** voice model owns turn-taking and TTS, and
the brain is a consulted specialist invoked only on hard turns. Keep it that way.

---

## Noise & turn-detection for speakerphone

Contractors call from job sites on speakerphone. Tiny noises (a mouse move, the
phone shifting) can false-trigger turn detection and cut the agent off
mid-sentence. The live levers in 0.6.3:

| Lever | Field | Default | For the noisy line |
|---|---|---|---|
| Barge-in floor (Realtime + Pipeline) | `barge_in_threshold_ms` / `bargeInThresholdMs` | `300` | Raise to **~500** so a transient doesn't count as a barge-in. `0` disables barge-in entirely. |
| Echo cancellation (**Pipeline only**) | `echo_cancellation` / `echoCancellation` | `false` | Set **`True`** for speakerphone/tunnel — stops the agent's own TTS bleed from holding VAD in "speaking" state. |
| Pipeline VAD threshold | `Agent(vad=SileroVAD.load(...))` | `activation_threshold=0.5`, `min_silence_duration=0.4` | Raise activation to **~0.8** (filters background noise) and silence to **~0.6-1.0 s** (lets callers pause mid-thought). |

```python
from getpatter import Patter, Twilio
from getpatter.providers.silero_vad import SileroVAD

phone = Patter(carrier=Twilio(), phone_number="+15550001234")
agent = phone.agent(
    # Pipeline mode (no engine=) — exposes the VAD + AEC levers
    system_prompt="You are the after-hours receptionist for Acme Roofing.",
    first_message="Thanks for calling Acme Roofing, how can I help?",
    barge_in_threshold_ms=500,                       # noisy speakerphone
    echo_cancellation=True,                          # pipeline-only; speakerphone bleed
    vad=SileroVAD.load(activation_threshold=0.8, min_silence_duration=0.7),
    consult=ConsultConfig(url="http://127.0.0.1:8000/consult", timeout_s=75.0, allow_loopback=True),
)
```

```typescript
import { Patter, Twilio } from "getpatter";
import { SileroVAD } from "getpatter";

const phone = new Patter({ carrier: new Twilio(), phoneNumber: "+15550001234" });
const agent = phone.agent({
  systemPrompt: "You are the after-hours receptionist for Acme Roofing.",
  firstMessage: "Thanks for calling Acme Roofing, how can I help?",
  bargeInThresholdMs: 500,                           // noisy speakerphone
  echoCancellation: true,                            // pipeline-only; speakerphone bleed
  vad: await SileroVAD.load({ activationThreshold: 0.8, minSilenceDuration: 0.7 }),
  consult: { url: "http://127.0.0.1:8000/consult", timeoutMs: 75_000, allowLoopback: true },
});
```

<Warning>
**Trade-off vs the long-call recipe.** Pipeline gives you `echo_cancellation` and
Silero VAD knobs, but **reassurance is Realtime-only** (above). You can't have
both first-class today: for the receptionist line, prefer **Realtime + raised
`barge_in_threshold_ms`** so reassurance works, and accept that the finer
server-VAD knobs (`far_field` noise reduction, server-VAD `threshold` /
`silence_duration_ms`) are not yet exposed in 0.6.3 — they are a pending SDK
addition. Do not promise the full server-VAD preset until it ships.
</Warning>

---

## Inbound: anyone can call

Patter answers **any** caller on its DID — no allowlist. There are two ways to
set the inbound agent:

- **In the SDK:** the agent you pass to `phone.serve(agent)` answers all inbound
  calls. Verify `X-Twilio-Signature` / Telnyx Ed25519 at the carrier edge
  (Patter does this for you) — that is the only trust boundary the public touches.
- **Via `patter-mcp` (Direction A):** the `configure_inbound` tool sets the
  default agent that answers all future inbound calls. Fields: `systemPrompt`,
  `firstMessage` (opt.), `engineMode` (default `pipeline`), `sttProvider` /
  `llmProvider` / `ttsProvider` (pipeline), `voice`, `language`.

Contrast with OpenClaw's native voice plugin, whose `inboundPolicy` is
allowlist-only (disabled by default) — it structurally cannot front random after-hours callers.
**Patter terminates the carrier leg for everyone; OpenClaw stays the brain
behind it.**

---

## Least-privilege agent scoping (OpenClaw side)

The voice layer must reach **only** the receptionist, never the master agent.
Model the receptionist as a **top-level** `agents.list[]` entry (not a sub-agent
of the master — sub-agent auth can fall back to the parent's credentials), with:

- its own workspace / `agentDir` / auth profiles,
- a tight `tools.allow` (only the calendar + customer-DB tools) and explicit
  `tools.deny` (exec / write / browser / gateway),
- per-agent sandbox (`mode: all`, `scope: agent`).

Expose any third-party MCP tools (calendar, customer DB) **only** to that agent
(per-server agent projection + a narrow `toolFilter.include`). Per-client
acceptance gate before going live:

1. List agents and their bindings; confirm the receptionist is bound to the
   inbound path and the master is not reachable from it.
2. Inspect the receptionist's **effective** tool list — it CAN reach the calendar
   / DB tools, and CANNOT reach exec / financial tools.
3. Run OpenClaw's security audit; verify the gateway binds loopback (not public).
4. Run **one full long-tool-call** end to end (a 45-60 s lookup) and confirm the
   call survives and the agent speaks the result.

> The exact OpenClaw config keys, CLI verbs, and audit commands are OpenClaw-side
> — confirm them against [docs.openclaw.ai](https://docs.openclaw.ai). Patter's
> contract is only the consult HTTP body and the named-agent `model` value.

---

## Direction A — OpenClaw places calls via Patter (`patter-mcp`)

Use this when OpenClaw should **initiate** calls. Run the MCP server from a local
clone (it is not on npm) and register it in `~/.openclaw/openclaw.json`. Prefer a
**long-lived streamable-http** server on an always-on box — OpenClaw reaps idle
stdio runtimes, which would re-spawn the embedded Patter server per session.

```json5
{
  mcp: {
    servers: {
      patter: {
        url: "http://localhost:3000/mcp",
        transport: "streamable-http",
        timeout: 600,                          // blocking make_call runs the whole call
        toolFilter: {
          // Narrow to the phone path — drop configure_inbound / get_metrics here.
          include: ["make_call", "call_third_party", "get_transcript", "get_calls", "end_call"],
        },
        // Scope this server to the receptionist via the agent's per-agent tool
        // policy (agents.list[].tools.allow + sandbox alsoAllow of "bundle-mcp").
        // NOTE: mcp.servers.<name>.codex.agents projects ONLY to the Codex
        // app-server runtime, NOT embedded-OpenClaw agents — don't rely on it here.
      },
    },
  },
  tools: {
    sandbox: {
      tools: {
        // MCP tools live in the "bundle-mcp" group; allowlist it (or "patter__*").
        alsoAllow: ["bundle-mcp"],
      },
    },
  },
}
```

The seven `patter-mcp` tools: `make_call`, `call_third_party`, `get_calls`,
`get_transcript`, `end_call`, `get_metrics`, `configure_inbound`. With
`wait: true`, `make_call` / `call_third_party` block until the call ends and
return the outcome plus transcript in one response.

Prefer OAuth / mTLS over static env-var secrets committed in the config, and
verify reachability after wiring (`openclaw mcp doctor patter --probe`).

---

## Gotchas

- **Never set `model: "openclaw"` / `"openclaw/default"` from the voice layer.**
  It resolves to the *default* agent (possibly the master) and drifts between
  environments. Always pin `"openclaw/<receptionistAgentId>"`.
- **The gateway credential is operator-grade.** The `model` field selects a
  persona, not a permission scope. Isolation = least-privileged agent + loopback
  binding + one-gateway-per-client, not the token.
- **Consult timeout defaults to 30 s** — too short for 30-60 s automation. Set
  `timeout_s` / `timeoutMs` to **75-90 s** AND raise the OpenClaw gateway timeout.
  The call dies at the shortest link.
- **Reassurance is Realtime-only.** Pipeline ignores it. Prefer Realtime for the
  receptionist line so the "let me check" filler actually plays.
- **Python has no per-tool timeout yet** — route slow work through `consult`
  (`timeout_s`), not a webhook tool (fixed 10 s). TS has `ToolDefinition.timeoutMs`.
- **`allow_loopback` is the intended shape here**, not a hack — on a co-located
  Mac Mini the adapter and gateway are on loopback. It relaxes only the consult
  URL's host check; non-HTTP(S) schemes are still rejected.
- **The adapter must forward `call_id` / `caller`.** The default doc adapter
  drops them; without `user=call_id` every consult starts a fresh OpenClaw
  session (no continuity) and caller-ID prefetch is impossible.
- **`far_field` noise reduction and server-VAD `threshold`/`silence_duration_ms`
  are NOT exposed in 0.6.3.** Today's live noise levers are
  `barge_in_threshold_ms`, `echo_cancellation` (pipeline), and Silero VAD
  (pipeline). Don't write config that sets non-existent fields.
- **`patter-mcp` is not on npm.** Run it from a local clone of
  [`PatterAI/patter-mcp`](https://github.com/PatterAI/patter-mcp).

## Common errors

| Symptom | Fix |
|---|---|
| Call drops ~30 s into a tool/lookup | Consult/adapter timeout still at the 30 s default, or the OpenClaw gateway/MCP timeout is lower. Raise them to cover the work and probe with `openclaw mcp doctor --probe` to confirm the per-server timeout is honoured. |
| Caller hears dead air during a lookup | No reassurance firing. Use Realtime mode and either set `reassurance` on a slow tool or prompt the agent to say "let me check" before calling `consult_agent`. |
| Consult reaches the wrong / a privileged agent | Adapter sends `model: "openclaw"`. Pin `"openclaw/<receptionistAgentId>"` and lock the agent's `tools.allow`/`deny`. |
| `ValueError: ConsultConfig url must be http(s)` / host rejected | Loopback/private host with the SSRF guard on. Set `allow_loopback=True` / `allowLoopback: true` for your local adapter URL. |
| Agent cut off mid-sentence on a noisy line | Raise `barge_in_threshold_ms` to ~500; in Pipeline also set `echo_cancellation=True` and raise the Silero `activation_threshold`. |
| Multi-turn call has no memory of earlier turns | Adapter not sending `user=call_id` — every consult opens a new OpenClaw session. |
| OpenClaw agent can't see Patter's MCP tools (Direction A) | `bundle-mcp` not allowlisted in `tools.sandbox.tools.alsoAllow`, or `timeout` too low for a blocking call. |
| Inbound calls from unknown numbers rejected | You're using OpenClaw's native voice plugin (allowlist-only). Put Patter on the DID instead; it answers everyone. |

## Related skills

- [`setup-patter`](../setup-patter/) — install the SDK and provision provider/carrier keys first.
- [`build-voice-agent`](../build-voice-agent/) — define the in-call agent that does the consulting.
- [`add-tools-and-handoffs`](../add-tools-and-handoffs/) — custom tools, reassurance, guardrails, `transfer_call` / `end_call`.
- [`configure-telephony`](../configure-telephony/) — wire the Twilio / Telnyx inbound webhook.
- [`inspect-calls-and-metrics`](../inspect-calls-and-metrics/) — confirm consult latency and call outcomes.

## References

- Patter OpenClaw integration: <https://docs.getpatter.com/integrations/openclaw>
- Patter consult feature: <https://docs.getpatter.com/python-sdk/consult> · <https://docs.getpatter.com/typescript-sdk/consult>
- Patter tools (reassurance, per-tool timeout): <https://docs.getpatter.com/python-sdk/tools> · <https://docs.getpatter.com/typescript-sdk/tools>
- Patter MCP server: <https://github.com/PatterAI/patter-mcp>
- OpenClaw docs (gateway, agents, MCP): <https://docs.openclaw.ai>
