## Context

oMLX currently has most of the technical building blocks for audio model modes:

- `model_discovery.py` can classify `audio_stt`, `audio_tts`, and `audio_sts` model types from `config.json`, TTS/STT mlx-audio remapping tables, STS mlx-audio model directories, architecture names, and static fallbacks.
- `EnginePool` stores audio model entries and can instantiate `STTEngine`, `TTSEngine`, and `STSEngine`.
- `api/audio_routes.py` exposes dedicated OpenAI-style audio endpoints for transcription, speech synthesis, and audio processing.
- The Admin model settings UI already includes audio model type override options.
- Packaging has an `audio` extra based on `mlx-audio[tts,stt,sts]`.

The gap is not a missing engine from scratch. The gap is making TTS and STS reliable, visible, and documented as first-class modes with clear user-facing behavior and well-defined failure modes.

The original task says `tts` and `sst`. This design scopes `sst` to `STS` because the repository uses `STS` for speech-to-speech/audio processing. It does not change STT behavior except where shared audio documentation or discovery context needs to mention it.

## Goals / Non-Goals

**Goals:**

- Make `audio_tts` and `audio_sts` explicit first-class model modes.
- Keep mode detection based on model metadata first, with controlled name-based fallbacks only where the current code already uses them.
- Ensure `/v1/models` continues listing TTS and STS model IDs, while `/v1/models/status` and Admin surfaces expose their correct model and engine types.
- Ensure requests to dedicated audio endpoints load the correct engine through `EnginePool`.
- Ensure requests to chat/completions/responses with TTS or STS models fail with clear endpoint guidance instead of unhandled engine errors.
- Ensure Admin model type override can force or reset TTS/STS classification.
- Ensure documentation explains installation, discovery, endpoint usage, and limitations.
- Add focused tests around discovery, routing, wrong-endpoint behavior, dependency-missing behavior, and memory lifecycle.

**Non-Goals:**

- Do not route TTS or STS through `/v1/chat/completions` or `/v1/responses`.
- Do not add a generic multimodal conversation protocol for audio in this change.
- Do not implement new mlx-audio model families beyond those supported by current mlx-audio loaders and oMLX family detection.
- Do not redesign LLM/VLM/embedding/reranker scheduling.
- Do not make `mlx-audio` a hard dependency for core-only installs.

## Decisions

### Keep dedicated audio endpoints

Use `/v1/audio/speech` for TTS and `/v1/audio/process` for STS/audio processing. This matches the existing router and keeps request/response formats aligned with audio payloads instead of forcing binary audio through chat APIs.

Alternative considered: expose TTS/STS through chat or responses. This would blur model mode semantics, complicate streaming/binary responses, and increase compatibility risk for existing text clients.

### Use `audio_tts` and `audio_sts` as canonical internal mode names

The repository already uses `audio_tts` and `audio_sts` in discovery, engine entries, EnginePool dispatch, and Admin override. The proposal should preserve these names instead of introducing aliases such as `tts` or `sst` internally.

Alternative considered: rename internal modes to `tts` and `sts`. This would create unnecessary migration churn and inconsistency with existing tests and data structures.

### Treat `STS` as speech-to-speech plus audio processing

The current `STSEngine` supports DeepFilterNet, MossFormer2, SAMAudio, and LFM2-style models. Some are speech enhancement or separation rather than pure speech-to-speech. The user-facing description should be "STS/audio processing" until a narrower product boundary is chosen.

Alternative considered: only support LFM2 speech-to-speech under STS. That would leave already-supported enhancement/separation families in an unclear state.

### Metadata-first discovery with cautious fallbacks

Continue preferring architecture and `model_type` values from `config.json`. Use mlx-audio remapping/directory sets when installed, and static fallback sets when it is not. Keep collision guards so text LLMs such as Qwen/Llama variants are not misclassified as audio.

Alternative considered: model-name-only detection. This is brittle and would misclassify text, embedding, or VLM models that happen to contain audio-like strings.

### Always mount audio routes and fail audio requests with install guidance when dependencies are missing

Core oMLX must remain importable without `mlx-audio`, but the audio route table should be stable. The implementation should always mount `/v1/audio/*` routes, keep lazy imports inside engines, and return an actionable `omlx[audio]` install error when an audio request needs missing runtime dependencies.

Alternative considered: keep the current hidden-route behavior when `mlx-audio` is unavailable. That keeps core startup simple but gives users a generic 404 instead of explaining which optional dependency is missing.

Alternative considered: require `mlx-audio` globally. This would simplify routing but make core installs heavier and more fragile.

### Defer STS family-specific request parameters

Keep `/v1/audio/process` scoped to `file` and `model` for this change. Supported STS families keep their existing engine defaults. Family-specific knobs such as SAMAudio `descriptions` or LFM generation limits can be added in a later change once the API contract is designed.

Alternative considered: add family-specific fields now. That would improve control for some STS models, but it expands the API surface beyond making the model mode first-class.

## Risks / Trade-offs

- [Risk] The task typo `sst` could mean `STT`, not `STS`. -> Mitigation: scope this change explicitly to STS/audio processing; if STT was intended, update the change before implementation or create a separate STT follow-up.
- [Risk] mlx-audio model metadata changes faster than oMLX static fallbacks. -> Mitigation: keep dynamic mlx-audio detection when installed and cover fallbacks with tests.
- [Risk] Some audio model families do not expose uniform generation parameters. -> Mitigation: keep this change limited to existing common endpoint parameters and defer family-specific extensions.
- [Risk] Audio endpoints are currently mounted only when `mlx-audio` imports successfully. -> Mitigation: move to always-mounted audio routes with dependency-specific errors and test core-only startup.
- [Risk] TTS streaming differs by model family. -> Mitigation: preserve current fallback behavior: native streaming when available, segmented WAV streaming otherwise.
- [Risk] STS `/v1/audio/process` currently accepts only `file` and `model`, while some families have useful parameters such as `descriptions` or generation limits. -> Mitigation: defer family-specific request parameters and document the current default behavior.

## Migration Plan

No data migration is expected. Existing model settings that already use `audio_tts` or `audio_sts` remain valid.

Implementation can be rolled out in place:

1. Tighten and test discovery/override behavior.
2. Tighten and test endpoint routing and error messages.
3. Add or update documentation and examples.
4. Run unit tests in a core-only environment and audio-enabled tests where MLX dependencies are available.

Rollback is straightforward: revert this change's code/docs/tests. No persistent format version bump is required unless implementation discovers that model settings serialization needs a schema change.

## Open Questions

None for this proposal. `/v1/models` remains OpenAI-compatible and lists model IDs; `/v1/models/status` and Admin surfaces carry the mode-specific details.
