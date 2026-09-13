---
name: choose-models-and-voices
description: Pick the STT, LLM and TTS providers and models for a Talkif flow from the live catalogue — GET /models and GET /models/tts/voices, latency vs quality vs cost trade-offs for phone audio, per-provider settings, ElevenLabs v3 vs flash, and a safe default. Use when building or tuning a flow's services block.
---

# Choose models and voices

## Read the catalogue, never hard-code

`GET /models` (public, `?useCase=` optional) → every STT/LLM/TTS provider and model with capabilities (e.g. whether an LLM accepts `temperature`), languages and current prices. `GET /models/tts/voices?search=&gender=&age=&language=&accent=&category=&voiceType=&pageSize=&page=` → voices with `voiceId`. Models get retired; a flow naming a retired model fails validation with `invalid_model`.

```typescript
const catalogue = await client.aiModels.listProvidersPublic();
const voices = await client.aiModels.listTtsVoices({ language: "en", gender: "female", search: "warm" });
```

## What each choice decides

| Service | Decides | Adds latency | Billed |
|---|---|---|---|
| STT | whether names, numbers, accents arrive correctly — errors here propagate | small, streams while the caller talks | per minute |
| LLM | what the agent says/does; prompt adherence | the largest share (time to first token) | per token, every turn resends prompts + transcript |
| TTS | how it sounds | time to first audio | per character |

Choose in that order. Target < ~1 s from the caller finishing to the agent starting.

## STT

Pick a model that lists the caller's language; auto-detecting models (Soniox, xAI) for mixed lines. Newer conversational models (e.g. Deepgram `nova-3-general`) handle spelled names and digit strings better. Telephony audio is 8 kHz — every listed model works; some advertise telephony tuning. Settings: `provider`, `model`, `language` (or `languages[]` hints for Soniox), `keywords[]` boosting where supported.

## LLM

Speed beats brilliance on the phone. Start with a small fast model (`gpt-4o-mini` class); move up only for multi-step reasoning inside a turn or strict policy compliance. Reasoning models add thinking time before every reply — keep `reasoningEffort` at the lowest the model accepts, or avoid them for live calls. Settings: `temperature` 0.3–0.7 (models that reject it have it stripped), `maxTokens` a few hundred (safety net, not style), `priorityTier: true` (OpenAI, 2× price, higher rate limits) for customer-facing lines under load.

## TTS

Listen before deciding — try two or three voices with the same prompt. Flash/low-latency models for conversation (`eleven_flash_v2_5`); expressive premium models where tone carries the brand. **ElevenLabs v3** (`eleven_v3`, `eleven_v3_conversational`) performs inline delivery tags (`[warmly]`) from the prompt, but only `stability` applies and tags appear in transcripts — see `write-prompts`. Settings by provider: ElevenLabs `stability similarity style speakerBoost speed`; Cartesia `emotion speed volume language`; Google/xAI/Rime `voiceId`, language, speed where offered.

## A default that works

```json
"services": {
  "stt": { "provider": "deepgram", "model": "nova-3-general", "language": "en" },
  "llm": { "provider": "openai", "model": "gpt-4o-mini", "temperature": 0.5, "maxTokens": 300 },
  "tts": { "provider": "elevenlabs", "model": "eleven_flash_v2_5", "voiceId": "<from /models/tts/voices>" }
}
```
Change one service at a time; compare time-to-first-response, interrupted turns and the LLM/TTS cost lines on the same test conversation.

Docs: https://docs.talkif.ai/build/choose-models-and-voices · https://docs.talkif.ai/reference/models-and-voices
