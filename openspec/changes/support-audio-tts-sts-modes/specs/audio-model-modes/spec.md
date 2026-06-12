## ADDED Requirements

### Requirement: TTS models are discoverable as audio_tts

The system SHALL classify supported text-to-speech model directories as `audio_tts` using model metadata before name-based fallbacks.

#### Scenario: TTS model detected by config model_type
- **WHEN** a model directory contains `config.json` with `model_type` set to `qwen3_tts`
- **THEN** model discovery reports `model_type` as `audio_tts`
- **AND** model discovery reports `engine_type` as `audio_tts`

#### Scenario: Text LLM is not misclassified as TTS
- **WHEN** a text LLM uses a base model type that collides with mlx-audio internals
- **THEN** model discovery MUST NOT classify it as `audio_tts` unless audio-specific architecture or model type evidence exists

### Requirement: STS models are discoverable as audio_sts

The system SHALL classify supported speech-to-speech and audio-processing model directories as `audio_sts` using model metadata before name-based fallbacks.

#### Scenario: STS model detected by architecture
- **WHEN** a model directory contains `config.json` with `architectures` containing `DeepFilterNetModel`
- **THEN** model discovery reports `model_type` as `audio_sts`
- **AND** model discovery reports `engine_type` as `audio_sts`

#### Scenario: LFM text model is not misclassified as STS
- **WHEN** a model directory uses an `lfm*` model type with a causal LM architecture
- **THEN** model discovery MUST classify it as an LLM instead of `audio_sts`

### Requirement: Audio models load through dedicated engines

The system SHALL instantiate TTS and STS models through their dedicated non-chat engines.

#### Scenario: Loading a TTS model
- **WHEN** a client requests an `audio_tts` model through the engine pool
- **THEN** the engine pool loads a `TTSEngine`

#### Scenario: Loading an STS model
- **WHEN** a client requests an `audio_sts` model through the engine pool
- **THEN** the engine pool loads an `STSEngine`

### Requirement: Dedicated audio endpoints serve TTS and STS modes

The system SHALL expose dedicated audio endpoints for TTS and STS/audio-processing requests when required audio runtime dependencies are available.

#### Scenario: TTS speech synthesis
- **WHEN** a client posts text input and an `audio_tts` model to `/v1/audio/speech`
- **THEN** the system returns audio bytes with `Content-Type` containing `audio/wav`

#### Scenario: STS audio processing
- **WHEN** a client posts an audio file and an `audio_sts` model to `/v1/audio/process`
- **THEN** the system returns processed audio bytes with `Content-Type` containing `audio/wav`

### Requirement: Model surfaces expose audio modes consistently

The system SHALL make TTS and STS models visible in model listing and status surfaces.

#### Scenario: Audio models listed by OpenAI-compatible models endpoint
- **WHEN** a TTS or STS model is discovered
- **THEN** `/v1/models` includes the model ID in its response data

#### Scenario: Audio model types visible in status surfaces
- **WHEN** a TTS or STS model is discovered
- **THEN** `/v1/models/status` and Admin model surfaces expose the model entry with `model_type` and `engine_type` set to `audio_tts` or `audio_sts`

### Requirement: Wrong endpoint usage returns actionable guidance

The system MUST reject TTS and STS models used through text-generation endpoints with an actionable endpoint hint.

#### Scenario: TTS model used for chat
- **WHEN** a client requests chat, completion, or responses generation with an `audio_tts` model
- **THEN** the system returns HTTP 400
- **AND** the response detail points the client to `/v1/audio/speech`

#### Scenario: STS model used for chat
- **WHEN** a client requests chat, completion, or responses generation with an `audio_sts` model
- **THEN** the system returns HTTP 400
- **AND** the response detail points the client to `/v1/audio/process`

### Requirement: Admin override supports TTS and STS modes

The system SHALL allow users to override model classification to `audio_tts` or `audio_sts` from persisted per-model settings and the Admin model settings UI.

#### Scenario: Override model to audio_tts
- **WHEN** a user sets a model type override to `audio_tts`
- **THEN** the model entry uses `audio_tts` as both model type and engine type

#### Scenario: Override model to audio_sts
- **WHEN** a user sets a model type override to `audio_sts`
- **THEN** the model entry uses `audio_sts` as both model type and engine type

#### Scenario: Reset override to auto
- **WHEN** a user clears the model type override
- **THEN** the system recomputes the model type from model discovery

### Requirement: Core installs remain usable without audio dependencies

The system SHALL keep core oMLX import and startup behavior usable when `mlx-audio` is not installed, while keeping audio routes mounted with actionable dependency errors.

#### Scenario: Core install discovers models without mlx-audio
- **WHEN** `mlx-audio` is unavailable
- **THEN** model discovery still runs using static audio detection fallbacks
- **AND** non-audio model serving remains available

#### Scenario: Audio request without runtime dependency
- **WHEN** a client posts to `/v1/audio/speech` or `/v1/audio/process` without required audio dependencies installed
- **THEN** the system returns HTTP 503
- **AND** the response detail explains that `omlx[audio]` or equivalent audio dependencies are required

### Requirement: Documentation describes TTS and STS model-mode usage

The system SHALL document TTS and STS as supported audio model modes.

#### Scenario: User looks for supported model modes
- **WHEN** a user reads `README.md`
- **THEN** the documentation lists TTS and STS/audio-processing alongside other supported model modes
- **AND** it explains the dedicated endpoints and audio extra installation requirement
