# Pipeline mode (STT → LLM → TTS)

Modular voice agent that wires together a Speech-to-Text provider, an LLM
provider, and a Text-to-Speech provider. Highest flexibility, every component
swappable. Use when you want a non-OpenAI voice, a custom LLM (Anthropic
Claude, Cerebras for speed, Groq for cost), or to mix-and-match providers
for cost optimization.

## When to use

- You want Anthropic Claude / Cerebras / Groq / Gemini as the brain.
- You want a specific voice (Cartesia Sonic, Rime, LMNT) not available in
  Realtime/ConvAI.
- You want the cheapest stack: Cerebras LLM + Deepgram STT + ElevenLabs Turbo
  TTS can run at < $0.05 / minute.

## When NOT to use

- You want lowest latency — Pipeline is ~600–800 ms turn time vs ~200 ms for
  Realtime. The three round-trips (STT→LLM→TTS) add up.
- You want the simplest setup — needs three keys minimum.

## Python (canonical low-cost stack)

```python
import asyncio
from getpatter import (
    Patter, Twilio,
    DeepgramSTT, CerebrasLLM, ElevenLabsTTS,
)

async def main():
    phone = Patter(carrier=Twilio(), phone_number="+15550001234")
    agent = phone.agent(
        # No engine= → pipeline mode
        stt=DeepgramSTT(model="nova-3"),
        llm=CerebrasLLM(model="gpt-oss-120b"),         # default in 0.5.4+
        tts=ElevenLabsTTS(voice_id="rachel"),
        system_prompt="You are a friendly receptionist.",
        first_message="Hi, how can I help?",
    )
    await phone.serve(agent, tunnel=True)

asyncio.run(main())
```

## TypeScript (canonical low-cost stack)

```typescript
import {
  Patter, Twilio,
  DeepgramSTT, CerebrasLLM, ElevenLabsTTS,
} from "getpatter";

const phone = new Patter({ carrier: new Twilio(), phoneNumber: "+15550001234" });

const agent = phone.agent({
  // No engine → pipeline mode
  stt: new DeepgramSTT({ model: "nova-3" }),
  llm: new CerebrasLLM({ model: "gpt-oss-120b" }),
  tts: new ElevenLabsTTS({ voiceId: "rachel" }),
  systemPrompt: "You are a friendly receptionist.",
  firstMessage: "Hi, how can I help?",
});

await phone.serve({ agent, tunnel: true });
```

## Provider matrix

### STT (Speech-to-Text)

| Provider | Class | Env | Notes |
|---|---|---|---|
| Deepgram | `DeepgramSTT` | `DEEPGRAM_API_KEY` | Default. Nova-3 is fast + accurate. |
| Whisper (OpenAI hosted) | `WhisperSTT` | `OPENAI_API_KEY` | Slower, batch-style. |
| OpenAI Transcribe | `OpenAITranscribeSTT` | `OPENAI_API_KEY` | Realtime transcribe API. |
| Cartesia | `CartesiaSTT` | `CARTESIA_API_KEY` | Sonic voice ecosystem. |
| Soniox | `SonioxSTT` | `SONIOX_API_KEY` | Multilingual specialist. |
| Speechmatics | `SpeechmaticsSTT` | `SPEECHMATICS_API_KEY` | Enterprise telephony. |
| AssemblyAI | `AssemblyAISTT` | `ASSEMBLYAI_API_KEY` | Universal model. |

### LLM

| Provider | Class | Env | Notes |
|---|---|---|---|
| OpenAI | `OpenAILLM` | `OPENAI_API_KEY` | `gpt-4o-mini` default. |
| Anthropic | `AnthropicLLM` | `ANTHROPIC_API_KEY` | Claude 3.5 Sonnet by default. |
| Cerebras | `CerebrasLLM` | `CEREBRAS_API_KEY` | `gpt-oss-120b` default (0.5.4+). Fastest. |
| Groq | `GroqLLM` | `GROQ_API_KEY` | `mixtral-8x7b` default. |
| Google | `GoogleLLM` | `GOOGLE_API_KEY` | Gemini 2.0 Flash. |

### TTS (Text-to-Speech)

| Provider | Class | Env | Notes |
|---|---|---|---|
| ElevenLabs (WebSocket) | `ElevenLabsTTS` / `ElevenLabsWebSocketTTS` | `ELEVENLABS_API_KEY` | Default in 0.6.1+. Streaming. |
| ElevenLabs (HTTP REST) | `ElevenLabsRestTTS` | `ELEVENLABS_API_KEY` | Slower, simpler. |
| OpenAI | `OpenAITTS` | `OPENAI_API_KEY` | `tts-1` default. |
| Cartesia | `CartesiaTTS` | `CARTESIA_API_KEY` | Sonic — sub-100ms first byte. |
| Rime | `RimeTTS` | `RIME_API_KEY` | English casual. |
| LMNT | `LMNTTTS` | `LMNT_API_KEY` | Fast, low-cost. |
| Inworld | `InworldTTS` | `INWORLD_API_KEY` | Game voice. |

## Tuning notes

- **Default low-latency stack**: Deepgram + Cerebras + ElevenLabs WebSocket.
  Cerebras runs at ~3000 tok/s on WSE-3 hardware, so `gpt-oss-120b` saturates
  the downstream TTS rate just as well as a smaller model — no latency
  penalty for using the bigger brain.
- **Cost-optimized stack**: Deepgram + Cerebras + LMNT — ~$0.03/min observed.
- **Quality-optimized stack**: AssemblyAI + Anthropic Claude 3.5 Sonnet +
  Cartesia Sonic — slower, more expensive, more natural.
- **Watch the per-leg latency** in `CallMetrics.latency_breakdown` to identify
  which leg is the bottleneck (e.g. if TTS first-byte is 400 ms, swap to
  Cartesia).

## Inbound audio quality & turn-taking

Pipeline mode exposes the full caller → STT audio chain. Every stage is opt-in
and the defaults keep audio byte-identical. Fixed stage order:

```
HPF (high_pass_hz) → resample → AEC → denoiser / audio_filter → AGC → VAD → STT
```

| Lever | Field (Py / TS) | Default | Use for |
|---|---|---|---|
| High-pass / DC-block (0.7.0) | `high_pass_hz` / `highPassHz` | off | Mains hum (50/60 Hz), handling rumble. Typical `80`–`120`. Runs first, before AEC. |
| Speech-selective AGC (0.7.0) | `agc` / `agc` (`bool` or `AgcConfig`) | `False` | Quiet or variable-distance talkers — normalises toward a target RMS. Silence is never amplified; a peak limiter prevents clipping. |
| Krisp denoiser by model id | `denoiser` / `denoiser` (string) | none | Job-site / speakerphone noise. BYO-license — see below. |
| Custom filter instance | `audio_filter` / `audioFilter` | none | Bring your own `AudioFilter` (e.g. `DeepFilterNetFilter`, `KrispVivaFilter`). Wins over `denoiser` when both are set. |
| Semantic end-of-turn | `turn_detector` / `turnDetector` | none | Stop cutting callers off mid-pause. See below. |

### Krisp denoiser by model id *(SDK main — next release after 0.7.0)*

Select a Krisp model with a stable string id instead of constructing a filter:

| Model id | Krisp session | Cancels |
|---|---|---|
| `krisp-viva-tel-v2` | NC | Telephony background noise (VIVA). |
| `krisp-bvc-o-pro-v3` | BVC | Background **voices** (other speakers near the caller). |

Krisp is **bring-your-own-license** — Patter ships zero Krisp binaries, license
keys, or `.kef` model files. The operator supplies all three:

1. Install the Krisp SDK: `pip install "getpatter[krisp]"` (Python) / your own
   licensed Node Krisp SDK (TypeScript).
2. Set `KRISP_VIVA_SDK_LICENSE_KEY` to your license key.
3. Set `KRISP_MODELS_DIR` to the directory holding your licensed `.kef` files
   (named `<model-id>.kef`, e.g. `krisp-viva-tel-v2.kef`).

```python
agent = phone.agent(
    stt=DeepgramSTT(model="nova-3"),
    llm=CerebrasLLM(model="gpt-oss-120b"),
    tts=ElevenLabsTTS(voice_id="rachel"),
    system_prompt="You are a friendly receptionist.",
    denoiser="krisp-viva-tel-v2",       # or "krisp-bvc-o-pro-v3"
    high_pass_hz=100,                    # hum + rumble ahead of the denoiser
    agc=True,                            # level the quiet talkers
)
```

```typescript
const agent = phone.agent({
  stt: new DeepgramSTT({ model: "nova-3" }),
  llm: new CerebrasLLM({ model: "gpt-oss-120b" }),
  tts: new ElevenLabsTTS({ voiceId: "rachel" }),
  systemPrompt: "You are a friendly receptionist.",
  denoiser: "krisp-viva-tel-v2",        // or "krisp-bvc-o-pro-v3"
  highPassHz: 100,
  agc: true,
});
```

Resolution is fail-fast at call start: a missing SDK, license key, models dir,
or `.kef` file raises a clear error naming exactly what to install or set — it
never silently degrades to a no-op. `denoiser` and `audio_filter` fill the same
chain slot; an explicit `audio_filter` instance wins.

### Semantic end-of-turn detection

By default the turn ends when VAD hears silence — which cuts off callers who
pause mid-sentence ("my email is john at… ‹pause› …gmail dot com"). Pass a
`turn_detector` and the handler holds the STT finalize until the model agrees
the turn is complete, bounded by `max_semantic_hold_ms` (default `1200`) so a
turn can never hang:

| Detector | Reads | Weights | Install |
|---|---|---|---|
| `SmartTurnDetector` | Audio prosody (last seconds of caller PCM) | smart-turn v3 ONNX ([pipecat-ai/smart-turn-v3](https://huggingface.co/pipecat-ai/smart-turn-v3)) — download, point `PATTER_SMART_TURN_MODEL` at it | `pip install "getpatter[turn-detector]"` / `onnxruntime-node` |
| `NamoTurnDetector` *(SDK main — next release after 0.7.0)* | Rolling transcript (last ~4 turns + in-flight utterance) | NAMO Turn Detector v1 ONNX (VideoSDK, Apache-2.0) — download the per-language model + HF tokenizer, point `PATTER_NAMO_MODEL` at the `.onnx` | `pip install "getpatter[namo-turn-detector]"` / `@huggingface/transformers` + `onnxruntime-node` |

```python
from getpatter import NamoTurnDetector

agent = phone.agent(
    stt=DeepgramSTT(model="nova-3"),
    llm=CerebrasLLM(model="gpt-oss-120b"),
    tts=ElevenLabsTTS(voice_id="rachel"),
    system_prompt="You are a friendly receptionist.",
    turn_detector=NamoTurnDetector.load(),   # reads PATTER_NAMO_MODEL
    max_semantic_hold_ms=1200,               # backstop — never hold longer
)
```

```typescript
import { NamoTurnDetector } from "getpatter";

const agent = phone.agent({
  stt: new DeepgramSTT({ model: "nova-3" }),
  llm: new CerebrasLLM({ model: "gpt-oss-120b" }),
  tts: new ElevenLabsTTS({ voiceId: "rachel" }),
  systemPrompt: "You are a friendly receptionist.",
  turnDetector: await NamoTurnDetector.load(),  // reads PATTER_NAMO_MODEL
  maxSemanticHoldMs: 1200,
});
```

NAMO ships **per-language models**, each with its own recommended decision
threshold — pick the model for your callers' language and pass
`threshold=` to `load()` (default `0.5`). The tokenizer is loaded from the
directory containing the model file, or from `PATTER_NAMO_TOKENIZER` /
`tokenizer_path=`. Weights are never bundled with the SDK.

## Common errors

| Symptom | Fix |
|---|---|
| `RuntimeError: no engine and no STT/LLM/TTS configured` | You forgot all three provider args. Add `stt=...`, `llm=...`, `tts=...`. |
| Agent talks but doesn't hear | STT provider key missing or wrong. Check `DEEPGRAM_API_KEY`. |
| Agent hears but doesn't talk | TTS provider key missing or wrong. Check `ELEVENLABS_API_KEY`. |
| Agent speaks slowly | Switch LLM to Cerebras or Groq (faster than OpenAI for short replies). |
| First TTS byte is slow | Switch TTS to Cartesia or ElevenLabs WebSocket (avoid REST/HTTP). |
| `Unknown denoiser '...'` | Only `krisp-viva-tel-v2` and `krisp-bvc-o-pro-v3` are registered. Check the id string. |
| `Denoiser ... needs the Krisp model directory` | Set `KRISP_MODELS_DIR` to the folder holding your licensed `.kef` files (and `KRISP_VIVA_SDK_LICENSE_KEY`). Krisp is BYO-license. |
| `NamoTurnDetector requires ...` ImportError | Install the extra: `pip install "getpatter[namo-turn-detector]"` / add `@huggingface/transformers` + `onnxruntime-node`. |
| NAMO model not found | Download a NAMO Turn Detector v1 `.onnx` + tokenizer and set `PATTER_NAMO_MODEL` (weights are not bundled). |
| Caller gets cut off mid-pause | Add a `turn_detector` (NAMO or smart-turn) — VAD-only endpointing treats any pause as end of turn. |
