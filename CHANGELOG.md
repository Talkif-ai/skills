# Changelog

All notable changes to the Talkif agent skills.

## 2026-11-13

### Added
- Gemini CLI extension (`gemini-extension.json`): `gemini extensions install https://github.com/Talkif-ai/skills` loads the skills and adds the Talkif MCP server, signed in with Talkif.
- `server.json` and a release workflow that lists the Talkif MCP server in the official MCP Registry as `io.github.Talkif-ai/talkif` when a `v*` tag is pushed.

## 2026-11-02

### Fixed
- `manage-phone-numbers` and `quickstart`: buying a phone number is dashboard-only — the account Owner buys it; `POST /phone/numbers/twilio/purchase` answers an API key with `403 api_key_not_allowed`. The skills now search by API, have the user buy in the dashboard, then find the number with `listPhoneNumbers` and connect it. `quickstart`'s key also gets `flow_templates:*`, which creating a flow from a template needs.
- `read-call-results`, `make-call` and `setup-api-key`: the call list is `calls.listCalls` / `list_calls` in SDK 0.3.0; `getCallHistory` and `getActiveCalls` were removed in 0.2.0. `listCalls` returns one page (`calls` + `meta`), so `read-call-results` now pages with `offset` instead of iterating a pager.
- `manage-phone-numbers`: Talkif has two SIP addresses — EU `sip.talkif.ai` / `63.176.211.13` and US `sip.us.talkif.ai` / `44.218.212.38`; outbound calls come from the provider's region, so carriers should allow both if unsure.
- `manage-phone-numbers`, `read-billing` and the README: free credit (including the welcome credit) can't pay for numbers only until the account's first payment; after it, any credit pays. `manage-phone-numbers` also says an unpaid number is released 14 days after it's suspended.
- `create-flow` and `choose-models-and-voices`: the example services used `eleven_flash_v2_5`, which Talkif retired on 2026-10-13 (publishing a flow with it fails), and `gpt-4o-mini` with a `temperature`. They now use `gpt-5.6-luna` (no `temperature`, which it rejects) and `eleven_v4_turbo`.
- `choose-models-and-voices`: ElevenLabs v4 Turbo replaces the retired v3 models in the TTS advice; the LLM advice no longer says unsupported settings are stripped (publishing rejects them) and drops `priorityTier`, which no current model offers.
- `write-prompts`: delivery tags are described for `eleven_v4_turbo`, the voice model that performs them now that v3 is retired.
- `update-flow`: the example switches to `gpt-5.6-luna` without `temperature`.
- `choose-models-and-voices`: the voice settings no longer list Google and Rime, which Talkif doesn't offer for new selections; the skill points to the live model list instead, and its description no longer offers "ElevenLabs v3 vs flash" (both retired).
