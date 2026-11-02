# Changelog

All notable changes to the Talkif agent skills.

## 2026-11-02

### Fixed
- `manage-phone-numbers` and `quickstart`: buying a phone number is dashboard-only — the account Owner buys it; `POST /phone/numbers/twilio/purchase` answers an API key with `403 api_key_not_allowed`. The skills now search by API, have the user buy in the dashboard, then find the number with `listPhoneNumbers` and connect it. `quickstart`'s key also gets `flow_templates:*`, which creating a flow from a template needs.
- `manage-phone-numbers`, `read-billing` and the README: free credit (including the welcome credit) can't pay for numbers only until the account's first payment; after it, any credit pays. `manage-phone-numbers` also says an unpaid number is released 14 days after it's suspended.
- `create-flow` and `choose-models-and-voices`: the example services used `eleven_flash_v2_5`, which Talkif retired on 2026-10-13 (publishing a flow with it fails), and `gpt-4o-mini` with a `temperature`. They now use `gpt-5.6-luna` (no `temperature`, which it rejects) and `eleven_v4_turbo`.
- `choose-models-and-voices`: ElevenLabs v4 Turbo replaces the retired v3 models in the TTS advice; the LLM advice no longer says unsupported settings are stripped (publishing rejects them) and drops `priorityTier`, which no current model offers.
- `write-prompts`: delivery tags are described for `eleven_v4_turbo`, the voice model that performs them now that v3 is retired.
- `update-flow`: the example switches to `gpt-5.6-luna` without `temperature`.
- `choose-models-and-voices`: the voice settings no longer list Google and Rime, which Talkif doesn't offer for new selections; the skill points to the live model list instead, and its description no longer offers "ElevenLabs v3 vs flash" (both retired).
