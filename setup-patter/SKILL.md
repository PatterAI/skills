---
name: setup-patter
description: >
  Install the Patter voice/telephony SDK (Python or TypeScript) and configure
  the provider and carrier API keys required for real phone calls. Use when
  the user is starting a new Patter project, hitting "missing API key" errors,
  setting up Twilio or Telnyx, integrating OpenAI Realtime / ElevenLabs /
  Deepgram, or asking how to get Patter running — even if they don't
  explicitly say "setup" or "credentials". Covers Patter 0.6.3 in both
  Python (>=3.11) and TypeScript (Node >=20) runtimes.
license: MIT
compatibility: >
  Requires Python 3.11+ or Node.js 20+. Needs at least one provider key
  (e.g. OPENAI_API_KEY) and one carrier credential set (Twilio
  TWILIO_ACCOUNT_SID + TWILIO_AUTH_TOKEN, or Telnyx TELNYX_API_KEY) in env.
  Patter >= 0.6.3.
metadata:
  author: patter
  version: "0.1.0"
  parity: both
---

# Set up Patter

Install the Patter SDK and provision the credentials required to place or
receive real phone calls. Patter is open-source — there is no Patter API key.
You bring your own provider (OpenAI / ElevenLabs / Deepgram / …) and your own
carrier (Twilio or Telnyx) keys, and Patter wires them together.

## Workflow

### Step 1 — Pick a runtime

Ask the user which SDK they want. Both have full feature parity.

- **Python** (3.11+): `pip install "getpatter>=0.6.3"`
- **TypeScript** (Node 20+): `npm install "getpatter@>=0.6.3"`

If they're unsure, default to whichever language the rest of their project
is in. Do not install both.

### Step 2 — Pick an engine (decides which provider keys are needed)

| Engine | What it does | Required keys |
|---|---|---|
| `OpenAIRealtime2` (recommended, GA in 0.6.3) | Speech-to-speech via OpenAI Realtime API — lowest latency | `OPENAI_API_KEY` |
| `OpenAIRealtime` (legacy) | Older `gpt-realtime-mini` model. Same key. | `OPENAI_API_KEY` |
| `ElevenLabsConvAI` | ElevenLabs ConversationAI — turn-taking model | `ELEVENLABS_API_KEY` |
| `Pipeline` (no engine arg) | STT → LLM → TTS — mix providers freely | At least one each of STT/LLM/TTS key |

If the user wants the simplest setup, default to **`OpenAIRealtime2`** — single
key, lowest latency, GA quality. Mention Pipeline only if they ask for custom
STT/LLM/TTS or for cost tuning.

### Step 3 — Pick a carrier

| Carrier | Audio format | Required env |
|---|---|---|
| **Twilio** | mulaw 8 kHz | `TWILIO_ACCOUNT_SID`, `TWILIO_AUTH_TOKEN` |
| **Telnyx** | PCM 16 kHz | `TELNYX_API_KEY` |

Twilio is the default suggestion — broader US/EU coverage, mature TwiML
ecosystem. Telnyx is preferable in regions where Twilio coverage is weak
and for users who want lower carrier cost.

Buying a phone number is **the user's job**, not the AI agent's — link the
console:

- Twilio: <https://console.twilio.com/us1/develop/phone-numbers/manage/incoming>
- Telnyx: <https://portal.telnyx.com/#/app/numbers/my-numbers>

### Step 4 — Get the keys (concierge the user through each console)

Patter is open-source — there is no Patter API key to issue. Every credential
is the user's own, obtained from the provider/carrier console. The agent's job
is to walk the user to each dashboard, one key at a time, and validate the
result before moving on.

Use the table below for the keys the user actually needs (based on the engine
and carrier picked in Steps 2 & 3). Skip rows that don't apply.

| Key | Dashboard | What to tell the user |
|---|---|---|
| `OPENAI_API_KEY` | <https://platform.openai.com/api-keys> | Click **+ Create new secret key** → name it (e.g. "patter-dev") → choose **All** permissions → copy the `sk-...` string — it's shown only once. |
| `TWILIO_ACCOUNT_SID` + `TWILIO_AUTH_TOKEN` | <https://console.twilio.com> | The Account SID and Auth Token are on the home dashboard under **Account Info**. Click **View** to reveal the token, then copy both. |
| `TELNYX_API_KEY` | <https://portal.telnyx.com/#/app/account/api-keys> | Click **Create API Key** → name it → copy the `KEY...` string. Then go to **Connections → SIP/SDK Connections** to grab `TELNYX_CONNECTION_ID`. |
| `ELEVENLABS_API_KEY` | <https://elevenlabs.io/app/settings/api-keys> | Click **Create API Key** → name it → choose scopes (`text_to_speech` minimum) → copy the `sk_...` string. |
| `DEEPGRAM_API_KEY` | <https://console.deepgram.com/project/_/keys> | Click **Create a New API Key** → role **Member** → copy. |
| `CEREBRAS_API_KEY` | <https://cloud.cerebras.ai/platform/api-keys> | Click **Generate API Key** → copy. |
| `ANTHROPIC_API_KEY` | <https://console.anthropic.com/settings/keys> | Click **Create Key** → copy. |
| `GOOGLE_API_KEY` | <https://aistudio.google.com/apikey> | Click **Create API key** → copy. |
| `GROQ_API_KEY` | <https://console.groq.com/keys> | Click **Create API Key** → copy. |

For each key, follow this concierge loop:

1. **Tell the user where to go** — paste the URL + the exact UI step from the
   table above. Wait for their reply.
2. **Receive the key** — they paste it in the chat. Treat it as sensitive: do
   NOT echo it back, do NOT commit it, do NOT log it.
3. **Validate before continuing** — a single cheap request that catches typos
   without burning the rate limit. Examples:

   ```bash
   # OpenAI — expect HTTP 200
   curl -s -o /dev/null -w "%{http_code}\n" https://api.openai.com/v1/models \
     -H "Authorization: Bearer $OPENAI_API_KEY"

   # Twilio — expect HTTP 200; account-info ping is free
   curl -s -o /dev/null -w "%{http_code}\n" \
     "https://api.twilio.com/2010-04-01/Accounts/$TWILIO_ACCOUNT_SID.json" \
     -u "$TWILIO_ACCOUNT_SID:$TWILIO_AUTH_TOKEN"

   # ElevenLabs — expect HTTP 200
   curl -s -o /dev/null -w "%{http_code}\n" https://api.elevenlabs.io/v1/user \
     -H "xi-api-key: $ELEVENLABS_API_KEY"
   ```

   If the status is `401` or `403`, the key is wrong — tell the user, ask them
   to re-paste, do not proceed. If it's `429`, the key is valid but rate-limited —
   accept it and continue.

4. **Only move to the next key** once the current one validated. This prevents
   silent failures later in Step 6 when `phone.serve()` raises `KeyError` and
   nobody knows which credential is broken.

### Step 5 — Write the keys to `.env`

Once every key is validated, write them to `.env` in the project root.
The agent should do this **after** Step 4, not before — pasting unvalidated
keys is the #1 source of "it doesn't work" support tickets.

```bash
# Engine
OPENAI_API_KEY=sk-...

# Carrier (pick one — Twilio shown)
TWILIO_ACCOUNT_SID=AC...
TWILIO_AUTH_TOKEN=...

# Phone number you bought from the carrier console
PATTER_PHONE_NUMBER=+15550001234
```

Verify `.env` is in `.gitignore`. If not, add it:

```
.env
```

**Never commit credentials.** Patter will pick these up automatically — every
carrier and provider reads its key from env by default.

### Step 6 — Smoke-test the install

Run the canonical 4-line example and verify it boots without ImportError or
KeyError. The agent does not have to place a real call — `serve(... tunnel=True)`
exiting cleanly with a tunnel URL is enough.

**Python** (`test_setup.py`):

```python
import asyncio
from getpatter import Patter, Twilio, OpenAIRealtime2

async def main():
    phone = Patter(carrier=Twilio(), phone_number="+15550001234")
    agent = phone.agent(
        engine=OpenAIRealtime2(),
        system_prompt="You are a friendly receptionist.",
        first_message="Hello!",
    )
    await phone.serve(agent, tunnel=True)  # Ctrl+C to stop once tunnel URL prints

asyncio.run(main())
```

**TypeScript** (`test-setup.ts`):

```typescript
import { Patter, Twilio, OpenAIRealtime2 } from "getpatter";

const phone = new Patter({ carrier: new Twilio(), phoneNumber: "+15550001234" });
const agent = phone.agent({
  engine: new OpenAIRealtime2(),
  systemPrompt: "You are a friendly receptionist.",
  firstMessage: "Hello!",
});
await phone.serve({ agent, tunnel: true });
```

Run with `python test_setup.py` or `tsx test-setup.ts`. Expected output: a
log line like `tunnel ready: https://<random>.trycloudflare.com` within ~5 seconds.

### Step 7 — Confirm and hand off

If smoke-test passes, tell the user:

> Patter 0.6.3 is set up. You can now use the `build-voice-agent` skill to
> design the agent, `configure-telephony` to wire the carrier webhook to the
> tunnel URL, or `add-tools-and-handoffs` to give the agent tools.

## Gotchas

- **`Patter(api_key=...)` raises `NotImplementedError`** in 0.6.3 — Patter
  Cloud was removed in 0.5.3 and will return as a future release. Always
  instantiate with `carrier=` + `phone_number=`, never with `api_key=`.
- **`pip install patter`** (no `get` prefix) installs an unrelated package.
  Always install `getpatter`.
- **Twilio kwargs in 0.6.3 normalize automatically** — `status_callback`,
  `machine_detection`, `timeout`, `async_amd` work as snake_case (Python) and
  camelCase (TS); no need to PascalCase them.
- **OpenAIRealtime vs OpenAIRealtime2** — both work in 0.6.3. Default to
  `OpenAIRealtime2` for new projects. `OpenAIRealtime` (model `gpt-realtime-mini`)
  is kept for back-compat.
- **Cloudflare `tunnel=True` is dev-only**. Production should use a static
  webhook URL (ngrok paid, or your own subdomain). The tunnel race on first
  call was fixed in 0.5.5 but a static URL is still more reliable for
  outbound campaigns.

## Common errors

| Symptom | Fix |
|---|---|
| `KeyError: OPENAI_API_KEY` | The env var isn't loaded. Source `.env` (`source .env` in bash, or use `python-dotenv` / `dotenv` package). |
| `twilio.base.exceptions.TwilioRestException: HTTP 401` | Wrong `TWILIO_AUTH_TOKEN`. Re-copy from console. |
| `RuntimeError: NotImplementedError: Patter Cloud …` | You passed `api_key=` to `Patter()`. Switch to `carrier=` + `phone_number=`. |
| TypeScript `Cannot find module 'getpatter'` | Wrong Node version (need 20+) or missed `npm install`. Check `node --version`. |
| `ModuleNotFoundError: No module named 'getpatter'` | Wrong Python venv active, or installed in the wrong interpreter. `pip list | grep getpatter`. |

## Related skills

- [`build-voice-agent`](../build-voice-agent/) — once setup is done, build the actual agent.
- [`configure-telephony`](../configure-telephony/) — full Twilio / Telnyx config beyond keys.

## References

- Patter Python quickstart: <https://docs.getpatter.com/python-sdk/quickstart>
- Patter TypeScript quickstart: <https://docs.getpatter.com/typescript-sdk/quickstart>
- Twilio console: <https://console.twilio.com>
- Telnyx portal: <https://portal.telnyx.com>
