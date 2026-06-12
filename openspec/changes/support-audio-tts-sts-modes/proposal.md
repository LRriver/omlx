## Why

oMLX already contains partial audio infrastructure for `audio_tts`, `audio_stt`, and `audio_sts`, but TTS and STS model modes are not yet documented as first-class supported capabilities. Users need a clear, reliable path from model discovery to model loading, API invocation, Admin override, and error handling for text-to-speech and speech-to-speech/audio-processing models.

## What Changes

- Define TTS and STS as first-class audio model modes in oMLX, using the existing `audio_tts` and `audio_sts` internal model type names.
- Ensure model discovery, per-model override, EnginePool loading, `/v1/models`, `/v1/models/status`, Admin model surfaces, and dedicated audio endpoints behave consistently for TTS and STS models.
- Document the expected user-facing APIs:
  - `POST /v1/audio/speech` for text-to-speech models.
  - `POST /v1/audio/process` for STS/audio-processing models.
- Preserve dedicated audio endpoints instead of routing TTS/STS through chat or responses APIs.
- Improve clarity around dependency behavior when `mlx-audio` is not installed.
- Clarify terminology: this change scopes the original `sst` wording to `STS` (speech-to-speech/audio processing). It does not change STT behavior except where shared audio documentation or discovery context needs to mention it.
- No breaking API changes are intended.

## Capabilities

### New Capabilities

- `audio-model-modes`: Defines first-class TTS and STS model-mode behavior across discovery, loading, API routing, Admin override, errors, and documentation.

### Modified Capabilities

- None.

## Impact

- Model discovery: `omlx/model_discovery.py`
- Engine lifecycle and memory management: `omlx/engine_pool.py`, `omlx/engine/tts.py`, `omlx/engine/sts.py`
- Audio APIs: `omlx/api/audio_routes.py`, `omlx/api/audio_models.py`
- Server routing and endpoint hints: `omlx/server.py`
- Admin model type override: `omlx/admin/routes.py`, Admin dashboard templates
- Packaging/install documentation and bundle flow for the existing `[audio]` extra; change `pyproject.toml` only if implementation discovers missing optional-extra metadata
- User documentation: README files and any audio/API docs
- Tests: audio discovery, audio endpoints, alias resolution, wrong-endpoint errors, dependency-missing behavior, and memory lifecycle tests
