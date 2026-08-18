# Repository Code Map — Parlor

This document provides a component-by-component index of all source files, classes, functions, and symbols within the **Parlor** repository.

---

## 1. Application Core (`src/parlor/`)

### 1.1. Server & Lifecycle Orchestrator
* **Location**: [server.py](file:///c:/Users/fadzw/parlor/src/parlor/server.py)
* **Purpose**: FastAPI web server initialization, WebSocket session handling, background task execution, mode management, and context window pruning.
* **Dependencies**: `fastapi`, `uvicorn`, `numpy`, `dotenv`, `parlor.llama`, `parlor.pipeline`, `parlor.actions`, `parlor.reasoner`, `parlor.tts`, `parlor.turn_detector`, `parlor.modes`.
* **Dependents**: Entrypoint `pyproject.toml:23` (`parlor = parlor.server:main`).
* **Key Symbols**:
  * `app`: `FastAPI` application instance with static mounts and lifespan context manager (`L312`).
  * `lifespan(app)`: Async context manager invoking model loading and cleanup (`L305`).
  * `load_models()`: Synchronously launches `llama-server`, loads ONNX turn detector, and initializes Kokoro TTS backend (`L297`).
  * `websocket_endpoint(ws)`: Main `/ws` WebSocket endpoint owning connection loop, history state, message queue, timer tasks, and turn execution (`L332`).
  * `rotate_history(history)`: Drops oldest 25% of user/assistant exchanges while preserving system prompt (`L274`).
  * `duration_phrase(seconds)` / `elapsed_phrase(seconds)`: Formats time durations into natural spoken phrasing (`L141`, `L249`).
  * System Prompts: `SYSTEM_PROMPT`, `CAPABILITY_NOTE`, `RESEARCH_NOTE`, `TRANSLATE_PROMPT`, `LISTEN_PROMPT`, `RESPOND_PROMPT`, `FLUSH_PROMPT`.

---

### 1.2. Streaming Turn Pipeline
* **Location**: [pipeline.py](file:///c:/Users/fadzw/parlor/src/parlor/pipeline.py)
* **Purpose**: Incremental SSE stream parsing, sentence extraction, audio silence padding, token estimation, and pipelined turn generation.
* **Dependencies**: `asyncio`, `base64`, `io`, `json`, `re`, `wave`, `numpy`, `parlor.llama`.
* **Dependents**: [server.py](file:///c:/Users/fadzw/parlor/src/parlor/server.py).
* **Key Symbols**:
  * `StreamParser`: Class incrementally parsing `###TRANSCRIPT: <words>\n<response>` and yielding complete response sentences (`L122`).
  * `run_turn(...)`: Async function orchestrating LLM decoding, sentence extraction, and Kokoro TTS audio chunk dispatching (`L263`).
  * `prime_cache(messages)`: Fire-and-forget request pushing prompt prefixes into `llama-server` KV cache (`L464`).
  * `pad_tail_silence(b64)`: Appends 300ms PCM silence inside audio WAV to prevent encoder truncation hallucinations (`L91`).
  * `echoes_instruction(transcript, instruction)`: Detects if model output echoes prompt text to suppress prompt leakage (`L234`).
  * `estimate_tokens(messages)`: Calculates heuristic token count across text, audio, and vision parts for context window management (`L66`).
  * Helper Functions: `image_part()`, `audio_part()`, `text_part()`, `valid_audio()`, `user_content()`, `wav_to_float32()`.

---

### 1.3. Decoupled Action Decider
* **Location**: [actions.py](file:///c:/Users/fadzw/parlor/src/parlor/actions.py)
* **Purpose**: Judgement of completed conversation turns via grammar-forced JSON LLM requests to execute system actions without in-band control markup.
* **Dependencies**: `json`, `dataclasses`, `parlor.llama`.
* **Dependents**: [server.py](file:///c:/Users/fadzw/parlor/src/parlor/server.py).
* **Key Symbols**:
  * `ActionDecision`: Dataclass holding typed action outputs (`timer`, `mode`, `research`) (`L87`).
  * `decide_after(messages, current_mode)`: Evaluates a completed conversational turn post-response (`L100`).
  * `decide_before(history, content, current_mode)`: Evaluates an utterance prior to response generation in listen/translate modes (`L111`).
  * `_decide(build, current_mode)`: Internal runner executing `llama.chat_blocking()` with `HEAD_SCHEMA` grammar constraint (`L125`).
  * `HEAD_SCHEMA`: JSON schema dictionary defining expected structure (`timer_seconds`, `timer_label`, `mode`, `research_task`) (`L73`).

---

### 1.4. Llama Server Subprocess & Client Driver
* **Location**: [llama.py](file:///c:/Users/fadzw/parlor/src/parlor/llama.py)
* **Purpose**: Subprocess management for `llama-server`, build version enforcement, model downloading, SSE streaming parsing, and REST client methods.
* **Dependencies**: `http.client`, `json`, `os`, `re`, `shutil`, `subprocess`, `huggingface_hub`, `dotenv`.
* **Dependents**: [server.py](file:///c:/Users/fadzw/parlor/src/parlor/server.py), [pipeline.py](file:///c:/Users/fadzw/parlor/src/parlor/pipeline.py), [actions.py](file:///c:/Users/fadzw/parlor/src/parlor/actions.py).
* **Key Symbols**:
  * `start()`: Spawns `llama-server` process with GGUF model and mmproj weights, polling `/health` (`L134`).
  * `stop()`: Terminates `llama-server` subprocess (`L167`).
  * `check_build(cmd, floor)`: Verifies `llama.cpp` binary build meets minimum version floor for Gemma 4 audio support (`L111`).
  * `chat_blocking(...)`: Non-streaming HTTP POST call returning complete text response (`L195`).
  * `ChatStream`: Thread-safe streaming class managing socket connection, reading SSE deltas, and extracting prompt token usage (`L213`).
  * `resolve_model_paths()`: Downloads GGUF and mmproj files via `huggingface_hub` (`L42`).
  * Model Registry: `MODELS` dict mapping `e2b`, `e4b`, `12b` to HuggingFace repositories (`L24`).

---

### 1.5. End-of-Turn Acoustic Classifier
* **Location**: [turn_detector.py](file:///c:/Users/fadzw/parlor/src/parlor/turn_detector.py)
* **Purpose**: Audio-native turn completeness classification using `smart-turn-v3.2` ONNX model and vendored NumPy log-mel feature extraction.
* **Dependencies**: `numpy`, `onnxruntime`, `huggingface_hub`.
* **Dependents**: [server.py](file:///c:/Users/fadzw/parlor/src/parlor/server.py).
* **Key Symbols**:
  * `TurnDetector`: Wrapper around `ort.InferenceSession` for `smart-turn-v3.2-cpu.onnx` (`L24`).
  * `predict(audio)`: Evaluates float32 16kHz audio array and returns `(complete: bool, probability: float)` (`L44`).
  * `compute_whisper_log_mel_features(audio)`: Vendored NumPy implementation replicating HuggingFace `WhisperFeatureExtractor` log-mel log filterbank calculation (`L179`).
  * `_power_spectrogram(waveform, ...)`: Batched real-FFT centered power spectrogram calculation (`L152`).

---

### 1.6. Unified Text-to-Speech Engine
* **Location**: [tts.py](file:///c:/Users/fadzw/parlor/src/parlor/tts.py)
* **Purpose**: Cross-platform abstraction interface for Kokoro-v1.0 TTS models.
* **Dependencies**: `numpy`, `platform`, `sys`, `mlx_audio` (optional), `kokoro_onnx` (optional), `huggingface_hub`.
* **Dependents**: [server.py](file:///c:/Users/fadzw/parlor/src/parlor/server.py).
* **Key Symbols**:
  * `TTSBackend`: Base class defining `generate(text, voice, speed)` interface (`L14`).
  * `MLXBackend`: Apple Silicon GPU backend utilizing `mlx-audio` (`L23`).
  * `ONNXBackend`: CPU fallback backend utilizing `kokoro-onnx` and HuggingFace assets (`L39`).
  * `load()`: Factory function inspecting environment and platform to return active `TTSBackend` (`L57`).

---

### 1.7. Background Research Assistant Client
* **Location**: [reasoner.py](file:///c:/Users/fadzw/parlor/src/parlor/reasoner.py)
* **Purpose**: HTTP client querying OpenAI-compatible external model endpoints (e.g. OpenRouter) for web-enabled background research.
* **Dependencies**: `urllib.request`, `json`, `os`.
* **Dependents**: [server.py](file:///c:/Users/fadzw/parlor/src/parlor/server.py).
* **Key Symbols**:
  * `enabled()`: Returns `True` if `REASONER_API_KEY` environment variable is configured (`L44`).
  * `ask(task)`: Synchronous blocking chat completion HTTP request executing system prompt for conversational text formatting (`L48`).

---

### 1.8. Session Operational Modes
* **Location**: [modes.py](file:///c:/Users/fadzw/parlor/src/parlor/modes.py)
* **Purpose**: Dataclass schema and static registry of operational session modes.
* **Dependencies**: `dataclasses`.
* **Dependents**: [server.py](file:///c:/Users/fadzw/parlor/src/parlor/server.py), [actions.py](file:///c:/Users/fadzw/parlor/src/parlor/actions.py).
* **Key Symbols**:
  * `Mode`: Immutable dataclass holding mode configuration flags (`name`, `uses_smart_turn`, `allows_delegation`, `wants_camera`, `wants_time_note`, `speaks_fallback`, `tts_voice`) (`L20`).
  * `MODES`: Registry dictionary containing `"conversation"`, `"translate"`, and `"listen"` instances (`L30`).

---

## 2. Frontend Interface (`src/parlor/web/`)

### 2.1. HTML Layout
* **Location**: [index.html](file:///c:/Users/fadzw/parlor/src/parlor/web/index.html)
* **Purpose**: Main web interface structure hosting viewport video canvas, audio waveform canvas, transcript view, status indicators, and control buttons.

### 2.2. Web Client JavaScript
* **Location**: [app.js](file:///c:/Users/fadzw/parlor/src/parlor/web/static/app.js)
* **Purpose**: Manages WebRTC camera stream, client Silero VAD (`vad-web`), AudioContext PCM output buffer queue, WebSocket message serialization, audio visualizer, and UI chip states.

### 2.3. Stylesheet
* **Location**: [style.css](file:///c:/Users/fadzw/parlor/src/parlor/web/static/style.css)
* **Purpose**: Dark-themed UI styling, glow canvas effects, responsive stacked layout, and animated status indicators.

---

## 3. Test & Benchmark Suite

### 3.1. Tests (`tests/`)
* [conftest.py](file:///c:/Users/fadzw/parlor/tests/conftest.py): Test setup, server fixtures, and audio generation mocks.
* [test_stream_parser.py](file:///c:/Users/fadzw/parlor/tests/test_stream_parser.py): StreamParser unit tests covering split token boundaries, instruction echo guards, and runaway line cuts.
* [test_conversation.py](file:///c:/Users/fadzw/parlor/tests/test_conversation.py): Multi-turn conversation integration tests.
* [test_delegation.py](file:///c:/Users/fadzw/parlor/tests/test_delegation.py): Background research delegation tests.
* [test_timer.py](file:///c:/Users/fadzw/parlor/tests/test_timer.py): Countdown timer scheduling and delivery tests.
* [test_robustness.py](file:///c:/Users/fadzw/parlor/tests/test_robustness.py): Interruption, audio glitch, and connection failure tests.

### 3.2. Benchmarks (`benchmarks/`)
* [archbench.py](file:///c:/Users/fadzw/parlor/benchmarks/archbench.py): Architecture benchmark measuring in-band control tags vs. decoupled action head accuracy.
* [turnbench.py](file:///c:/Users/fadzw/parlor/benchmarks/turnbench.py): Turn completeness benchmark comparing Smart Turn v3 against LLM-based turn gating.
* [bench.py](file:///c:/Users/fadzw/parlor/benchmarks/bench.py): End-to-end latency benchmark measuring TTFA (Time to First Audio).
