## 1. Scope Baseline

- [x] 1.1 Keep this change scoped to TTS plus STS/audio-processing; do not change STT behavior except shared audio documentation or discovery references.
- [x] 1.2 Change audio route registration so `/v1/audio/*` routes are always mounted and missing audio runtime dependencies return actionable `omlx[audio]` install errors.
- [x] 1.3 Keep `/v1/audio/process` request input limited to `file` and `model`; document that STS family-specific parameters are deferred.

## 2. Model Discovery and Overrides

- [x] 2.1 Audit `audio_tts` and `audio_sts` discovery paths for architecture-first and model-type-first classification.
- [x] 2.2 Update `tests/test_audio_discovery.py` or equivalent discovery tests for concrete TTS fixtures such as `model_type=qwen3_tts` and TTS architecture values.
- [x] 2.3 Update `tests/test_audio_discovery.py` or equivalent discovery tests for concrete STS fixtures such as `DeepFilterNetModel`, `SAMAudio`, and `LFM2AudioModel`.
- [x] 2.4 Update negative discovery tests that prevent text LLMs with colliding model types from being classified as TTS or STS.
- [x] 2.5 Implement or adjust Admin and persisted settings override tests so `audio_tts` and `audio_sts` can be set and reset consistently.

## 3. Engine Loading and Endpoint Routing

- [x] 3.1 Implement or adjust `EnginePool` dispatch and tests so `audio_tts` entries create `TTSEngine` and `audio_sts` entries create `STSEngine`.
- [x] 3.2 Update memory lifecycle tests for TTS and STS loading, unloading, last-access updates, and LRU eligibility.
- [x] 3.3 Implement or adjust `/v1/models`, `/v1/models/status`, and Admin model surfaces so TTS and STS model IDs are listed and their `model_type`/`engine_type` details are visible where supported; add or update tests.
- [x] 3.4 Implement or adjust `/v1/audio/speech` tests so `audio_tts` requests return WAV audio for non-streaming requests.
- [x] 3.5 Implement or adjust `/v1/audio/speech` streaming tests for native streaming and segmented fallback paths.
- [x] 3.6 Implement or adjust `/v1/audio/process` tests so `audio_sts` requests return WAV audio using the current `file` plus `model` API.

## 4. Errors and Dependency Behavior

- [x] 4.1 Implement and test chat/completions/responses rejection for `audio_tts` models with HTTP 400 and `/v1/audio/speech` guidance.
- [x] 4.2 Implement and test chat/completions/responses rejection for `audio_sts` models with HTTP 400 and `/v1/audio/process` guidance.
- [x] 4.3 Implement and test always-mounted `/v1/audio/*` routes in core installs without `mlx-audio`, returning HTTP 503 with `omlx[audio]` install guidance for audio requests.
- [x] 4.4 Ensure TTS and STS engine import/load failures mention `omlx[audio]` or equivalent install guidance.

## 5. Documentation

- [x] 5.1 Update primary README documentation to list TTS and STS/audio-processing as supported model modes.
- [x] 5.2 Document `pip install -e ".[audio]"` or equivalent audio extra installation.
- [x] 5.3 Add examples for `POST /v1/audio/speech` and `POST /v1/audio/process`.
- [x] 5.4 Update localized README files (`README.zh.md`, `README.fr.md`, `README.ja.md`, `README.ko.md`) with matching audio-mode summaries.
- [x] 5.5 Review packaging documentation for the existing `[audio]` extra and update it when implementation changes install or bundle behavior.

## 6. Verification

- [ ] 6.1 Run focused unit tests for audio discovery, audio API routes, Admin override, and EnginePool memory lifecycle.
- [ ] 6.2 Run a core-only import/startup check without `mlx-audio` if the local environment allows dependency isolation.
- [x] 6.3 Run audio-enabled integration tests or document why they were skipped when MLX/audio dependencies are unavailable.
- [x] 6.4 Run `openspec validate support-audio-tts-sts-modes --strict`.

Verification notes:
- Passed: `python -m compileall omlx/engine/audio_utils.py omlx/api/audio_routes.py tests/test_audio_api.py tests/test_audio_tts.py tests/test_audio_sts.py tests/test_audio_memory.py tests/test_engine_pool.py tests/test_model_settings.py tests/test_audio_discovery.py tests/test_server.py`
- Passed locally: `python -m pytest tests/test_audio_discovery.py tests/test_model_settings.py tests/test_audio_tts.py::TestTTSEndpointErrors::test_missing_audio_dependency_returns_503 tests/test_audio_tts.py::TestTTSEndpointErrors::test_missing_audio_dependency_during_engine_import_returns_503 tests/test_audio_sts.py::TestSTSEndpointErrors::test_missing_audio_dependency_returns_503 tests/test_audio_sts.py::TestSTSEndpointErrors::test_missing_audio_dependency_during_engine_import_returns_503 tests/test_audio_api.py tests/test_audio_memory.py tests/test_engine_pool.py tests/test_server.py -q` (`103 passed, 4 skipped`).
- Passed: `openspec validate support-audio-tts-sts-modes --strict`
- Not completed locally: full focused tests for `/v1/models`, Admin override via `EnginePool`, EnginePool memory lifecycle, and server wrong-endpoint handling were skipped here because this Python environment does not have core MLX packages installed. These should be run in the project MLX environment before final archive/release.
- Not completed locally: core startup without `mlx-audio` could not be isolated because the current environment is missing core `mlx`, not only optional `mlx-audio`.
- Skipped: audio-enabled integration tests require MLX/audio runtime dependencies and model downloads.
