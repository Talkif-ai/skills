# Changelog

All notable changes to the Talkif agent skills.

## 2026-11-02

### Fixed
- `create-flow` and `choose-models-and-voices`: the example services used `eleven_flash_v2_5`, which Talkif retired on 2026-10-13 (publishing a flow with it fails), and `gpt-4o-mini` with a `temperature`. They now use `gpt-5.6-luna` (no `temperature`, which it rejects) and `eleven_v4_turbo`.
- `choose-models-and-voices`: ElevenLabs v4 Turbo replaces the retired v3 models in the TTS advice; the LLM advice no longer says unsupported settings are stripped (publishing rejects them) and drops `priorityTier`, which no current model offers.
- `write-prompts`: delivery tags are described for `eleven_v4_turbo`, the voice model that performs them now that v3 is retired.
- `update-flow`: the example switches to `gpt-5.6-luna` without `temperature`.
