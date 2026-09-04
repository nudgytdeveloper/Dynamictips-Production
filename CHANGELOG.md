# Changelog

## 2026-09-04

### Added
- Tips follow the learner's Continuous Learning instruction: the agent reads the active `agent_learner_instructions` document for the simulation's learner and injects it into the prompt after the SOP block; each persisted tip entry records `cl_instruction_id`, `cl_instruction_version` and `cl_stage`, and the `/tips` response gains `cl_applied` (ported from staging, 2026-08-24)
- `duration_ms` on the `/tips` response, and one `Tips for simulation …: <total>ms (lookup= context= gemini= save=)` log line per request
- `GEMINI_MODEL` (default `gemini-2.5-flash`), `GEMINI_THINKING_BUDGET` (default `0`), `GEMINI_TIMEOUT_SECONDS` (default `60`), `CL_INSTRUCTIONS_COLLECTION`, `SOP_CACHE_TTL_SECONDS` and `SOP_CACHE_FAILURE_TTL_SECONDS` env vars

### Changed
- Gemini is called with thinking switched off; the agent now uses the `google-genai` SDK instead of the end-of-life `google-generativeai` (ported from staging, 2026-09-01)
- The SOP file is downloaded once per URL and cached instead of on every request
- Transcript and simulation lookups run concurrently, as do the Continuous Learning lookup and the SOP download; the Continuous Learning audit writes run after the response is sent

### Fixed
- A Gemini response with no text (safety block, or the whole budget spent on thinking) is returned as an error payload instead of raising
- Log lines reach Render as they happen instead of in one clump at the end of the request
- Running the file directly (`python DynamicTips_Production.py`) pointed uvicorn at a module that does not exist

## 2026-08-11

### Added
- Tips are now grounded in the service's uploaded SOP: the agent resolves the simulation's `sop_tips_generation_file` from `dependency_snapshot` (falling back to the live `service_levels` doc), downloads the file, and passes a PDF to Gemini as inline data (or inlines a `.txt`/`.md` body), instructing the model to prefer a tip about an SOP instruction the learner did not follow over generic communication advice — task 307
- `SIMULATIONS_COLLECTION` and `SERVICE_LEVELS_COLLECTION` env vars (defaulting to `simulations` / `service_levels`) so the agent can resolve a simulation to its service's SOP
- `sop_applied` and `sop_file_url` on the `/tips` response and on each persisted tip entry, so a tip can be traced back to the document that produced it

### Fixed
- SOP file URLs are percent-encoded before download. Uploaded SOP filenames routinely contain spaces (e.g. `Tips SOP Sample .pdf`), which `urllib` rejects with `http.client.InvalidURL`; found by the first end-to-end run against a real staging simulation, which returned HTTP 500
- `fetch_sop_document` caught only `URLError`/`HTTPError`/`OSError`/`ValueError`, but `http.client.InvalidURL` descends from `Exception` alone, so a bad SOP URL escaped as an unhandled 500 instead of degrading. It now catches `Exception`, honouring its documented "never raises" contract
- Transcript turns were labelled `HUMAN:` / `AI:` while the prompt instructed the model to analyse the `HUMAN USER` messages and ignore the `AI CHARACTER` messages — labels that appeared nowhere in the transcript being sent. Now labelled `HUMAN USER` / `AI CHARACTER`, matching the prompt and the staging agent
- `generate_tips` returned `[]` on a generation failure while `TipsResponse.tips` is typed `str`, turning every Gemini error into a pydantic validation error (HTTP 500) instead of a clean error payload; it now returns `""`

### Changed
- A missing simulation, missing SOP field, unreachable URL, oversized file (>15 MB), or unsupported file type logs and falls back to the previous generic tip instead of failing the request
- The SOP upload UI shipped on staging first, so this is expected to be a no-op (`sop_applied: false`) on production until the platform promotes the feature
