# Repository Deep Review — Parlor

## Executive Summary

**Parlor** is an on-device, real-time multimodal AI application designed for low-latency voice and vision interaction. Built around local neural inference (using Google's Gemma 4 audio-language models via `llama-server`), ONNX-based acoustic end-of-turn detection (`smart-turn-v3.2`), and Kokoro speech synthesis, Parlor provides a local and private conversational experience.

Unlike conventional voice assistants that rely on cloud APIs or in-band control markup, Parlor decouples conversational speech from system actions (timer setting, background web research, session mode switching) through a separate, grammar-forced JSON action head. The entire system is orchestrated via an asynchronous FastAPI / WebSockets backend (`src/parlor/server.py`) and rendered through a web frontend using browser-side Voice Activity Detection (`@ricky0123/vad-web`).

---

## What This Application Does

Parlor enables users to converse with an AI assistant using speech, text, and camera input:
1. **Real-Time Voice & Vision Interaction**: Streams raw 16kHz audio and periodic JPEG camera frames over a persistent WebSocket connection.
2. **Acoustic Turn Gating**: Uses a 140k-parameter ONNX model (`smart-turn-v3.2`) to evaluate audio completion probability on CPU in tens of milliseconds, preventing premature interruptions during user speech pauses.
3. **Speculative Prompt-Cache Priming**: Warm-up requests stream mid-utterance audio and camera frames directly into `llama-server` context memory while the user is still speaking.
4. **Decoupled Action Decision**: Evaluates completed conversation turns using a secondary, JSON-constrained LLM request to trigger system actions (timers, background research, session mode switching) without polluting spoken outputs.
5. **Multi-Mode Operation**:
   - **Conversation**: Full bi-directional voice + vision assistant.
   - **Translate**: Consecutive speech-to-speech interpreter translating spoken input into English.
   - **Listen**: Silent scribe transcribing spoken thoughts without verbal responses until explicitly requested.
6. **Delegated Background Research**: Hands off current or web-search queries to an OpenAI-compatible external model endpoint (e.g., OpenRouter) while keeping primary interaction fully local.

---

## Problem It Solves

1. **Voice Latency & Interruption**: Traditional voice pipelines suffer from latency introduced by sequential STT → LLM → TTS pipelines. Parlor pipelines LLM decoding directly into sentence chunking and Kokoro TTS synthesis, while speculatively priming prompt KV caches.
2. **In-Band Tag Hallucination & Leakage**: Relying on an LLM to generate control tags (e.g., `<timer>120</timer>`) alongside conversational speech frequently results in tags being read aloud or omitted. Parlor isolates action logic into a grammar-forced head (`actions.py`).
3. **Privacy & Infrastructure Dependence**: Most voice assistants stream audio to cloud services. Parlor runs core inference locally on consumer hardware (Mac Silicon via MLX or CPU via ONNX Runtime).

---

## Technology Stack

* **Programming Language**: Python 3.12 (`pyproject.toml:5`)
* **Web Framework & Server**: FastAPI 0.135+, Uvicorn 0.43+, WebSockets 16.0 (`src/parlor/server.py:26`)
* **Local LLM Engine**: `llama.cpp` (`llama-server`) hosting Gemma 4 GGUF models (`e2b`, `e4b`, `12b`) (`src/parlor/llama.py:24`)
* **Acoustic EOT Classification**: `onnxruntime` executing `pipecat-ai/smart-turn-v3.2-cpu.onnx` (`src/parlor/turn_detector.py:15`)
* **Text-to-Speech (TTS)**: Kokoro-v1.0 via `mlx-audio` on macOS ARM64 (`src/parlor/tts.py:23`) or `kokoro-onnx` on CPU (`src/parlor/tts.py:39`)
* **Background Research Endpoint**: Python standard `urllib.request` against OpenRouter / OpenAI API (`src/parlor/reasoner.py:15`)
* **Frontend**: Vanilla HTML5 / JavaScript ES6 (`src/parlor/web/index.html`), `ONNX Runtime Web`, `@ricky0123/vad-web` (`src/parlor/web/index.html:56-57`)
* **Testing & Benchmarks**: Pytest 8.4 (`pyproject.toml:30`), custom benchmark suite in `benchmarks/`

---

## Repository Structure

```text
parlor/
├── .env.example            # Sample configuration environment variables
├── pyproject.toml          # Project dependencies, entrypoints, tool settings
├── uv.lock                 # Lockfile for reproducible environment setup
├── README.md               # User-facing project documentation
├── CHANGELOG.md            # Version release notes
├── LICENSE                 # Apache-2.0 License
├── docs/
│   └── configuration.md    # Detailed environment variable reference
├── src/
│   └── parlor/
│       ├── __init__.py      # Package indicator
│       ├── modes.py        # Data structures for session modes (conversation, translate, listen)
│       ├── reasoner.py     # Client for background web research model endpoints
│       ├── tts.py          # Unified TTS interface (mlx-audio / kokoro-onnx)
│       ├── actions.py      # Grammar-forced JSON action decision head
│       ├── turn_detector.py # Smart Turn v3.2 ONNX classifier & log-mel feature extractor
│       ├── llama.py        # Subprocess manager & HTTP/REST client for llama-server
│       ├── pipeline.py     # Turn execution pipeline, stream parsing, WAV utilities
│       ├── server.py       # FastAPI application, WebSocket router, turn loop, event handlers
│       └── web/
│           ├── index.html  # Application HTML interface template
│           └── static/
│               ├── app.js   # Client-side VAD, audio capture/playback, WebSocket protocol
│               └── style.css# UI stylesheet and design system
├── tests/                  # Unit and integration test suite
│   ├── conftest.py         # Pytest fixtures and mock servers
│   ├── test_conversation.py
│   ├── test_delegation.py
│   ├── test_elapsed.py
│   ├── test_history.py
│   ├── test_listen.py
│   ├── test_llama_startup.py
│   ├── test_robustness.py
│   ├── test_stream_parser.py
│   ├── test_timer.py
│   ├── test_transcription_stability.py
│   ├── test_translation.py
│   ├── test_z_failure.py
│   └── util.py
└── benchmarks/             # Evaluation and benchmarking scripts
    ├── archbench.py
    ├── bench.py
    ├── benchmark_tts.py
    ├── camerabench.py
    ├── compare.py
    ├── fixtures.py
    ├── legacy_tags.py
    ├── tagbench.py
    ├── timerprobe.py
    └── turnbench.py
```

---

## Architecture

The system operates as a single-process FastAPI application that manages `llama-server` as a subprocess.

```mermaid
graph TD
    Client["Web Browser (index.html / app.js)"] <-->|WebSocket /ws| Server["FastAPI Server (server.py)"]
    
    subgraph Local Inference Subsystems
        Server <-->|HTTP POST /v1/chat/completions| LlamaServer["llama-server Subprocess (llama.py)"]
        Server -->|In-Process ONNX| TurnDetector["Smart Turn v3 Classifier (turn_detector.py)"]
        Server -->|In-Process MLX/ONNX| TTSBackend["Kokoro TTS Engine (tts.py)"]
    end
    
    subgraph External Services
        Server -->|HTTP POST (urllib)| ReasonerAPI["OpenRouter / OpenAI Endpoint (reasoner.py)"]
    end
```

Key Architectural Principles:
1. **Single-Slot Prefix Caching**: `llama-server` is configured with `-np 1` (`src/parlor/llama.py:145`). Conversation history is re-sent every turn, benefiting from full prefix-cache hits.
2. **Grammar-Forced Structured Head**: Action decisions are decoupled into an auxiliary JSON-schema request (`src/parlor/actions.py:73-83`) sharing the same KV cache prefix.
3. **Pipelined Generation**: Text tokens stream from `llama-server` into `StreamParser` (`src/parlor/pipeline.py:122`), which yields complete sentences to `tts_worker` (`src/parlor/pipeline.py:311`) before full turn completion.

---

## Application Startup

Startup is managed by FastAPI's lifespan handler (`src/parlor/server.py:305-310`):

```text
CLI Entrypoint (parlor = parlor.server:main)
 ↓
FastAPI Lifespan Startup Hook
 ↓
load_models() [src/parlor/server.py:297]
 ├── 1. llama.start() [src/parlor/llama.py:134]
 │       ├── Locate binary (llama-server or llama serve)
 │       ├── Enforce minimum build version (MIN_BUILD >= 9503)
 │       ├── Download / locate GGUF & mmproj weights via HuggingFace Hub
 │       ├── Spawn llama-server subprocess (-c 16384 -ngl 99)
 │       └── Poll /health endpoint until HTTP 200 (up to 180s)
 ├── 2. TurnDetector() [src/parlor/turn_detector.py:24]
 │       ├── Download / locate smart-turn-v3.2-cpu.onnx
 │       └── Initialize ort.InferenceSession & run zero-vector warmup
 └── 3. tts.load() [src/parlor/tts.py:57]
         └── Initialize MLXBackend (macOS arm64) or ONNXBackend (fallback)
 ↓
FastAPI app ready to accept WebSocket connections at ws://localhost:8000/ws
```

---

## Primary Execution Flows

### User Speech Turn Flow
1. Client VAD detects end of speech segment and sends `speech_chunk` or `audio` payload over WebSocket (`src/parlor/web/static/app.js:250`).
2. Server runs `TurnDetector.predict()` on concatenated float32 PCM (`src/parlor/server.py:740`).
3. If `p_complete < 0.5`, server sends `turn_incomplete`, caches `held_audio`, and waits (`src/parlor/server.py:746`).
4. If `p_complete >= 0.5`, server appends tail silence (`src/parlor/pipeline.py:91`), builds messages, and invokes `run_turn()`.
5. `run_turn()` starts `llama.ChatStream` (`src/parlor/pipeline.py:292`) in a background thread.
6. `StreamParser` extracts the leading `###TRANSCRIPT:` line and streams response sentences to `tts_worker` (`src/parlor/pipeline.py:311`).
7. `tts_worker` generates PCM audio and streams `audio_start` / `audio_chunk` frames to the client (`src/parlor/pipeline.py:325`).
8. After turn completion, `actions.decide_after()` runs a grammar-constrained JSON request against `llama-server` (`src/parlor/server.py:820`).
9. `apply_decision()` executes any requested timers, mode switches, or background research tasks (`src/parlor/server.py:555`).

---

## Data Flow

```text
[Mic 16kHz PCM Audio]
 ↓ (Client VAD)
[Base64 WAV WebSocket Message]
 ↓
[server.py: websocket_endpoint]
 ↓
[turn_detector.py: compute_whisper_log_mel_features]
 ↓ (80x800 float32 matrix)
[smart-turn-v3.2 ONNX Session] → Output: p_complete
 ↓ (If complete)
[pipeline.py: user_content & pad_tail_silence]
 ↓ (Multimodal JSON Message Structure)
[llama.py: ChatStream POST /v1/chat/completions]
 ↓ (SSE Stream: "data: {...}")
[pipeline.py: StreamParser]
 ├── Transcript Delta → ws.send_json({"type": "transcript"})
 └── Sentence Delta → Queue → tts.py (generate) → ws.send_json({"type": "audio_chunk"})
```

---

## Important Components

1. **`src/parlor/server.py`**: Central coordinator. Manages WebSocket connections, session state (`history`, `mode`, `pending_timers`, `ready_events`), turn loop orchestration, and history pruning (`rotate_history`).
2. **`src/parlor/pipeline.py`**: Stream parsing (`StreamParser`), prompt token estimation (`estimate_tokens`), audio format conversion, and async pipelining (`run_turn`).
3. **`src/parlor/actions.py`**: Action decider. Formats JSON schema and system prompt for judging turns (`decide_after` / `decide_before`) with zero temperature.
4. **`src/parlor/llama.py`**: Process lifecycle manager for `llama-server`. Handles binary version verification, model downloading, SSE streaming parsing (`ChatStream`), and blocking REST calls (`chat_blocking`).
5. **`src/parlor/turn_detector.py`**: ONNX inference wrapper for turn completeness. Vendors Whisper log-mel spectrogram computation (`compute_whisper_log_mel_features`) in standard NumPy.
6. **`src/parlor/tts.py`**: Cross-platform TTS loader. Dynamically selects `mlx-audio` on Apple Silicon or `kokoro-onnx` on other platforms.
7. **`src/parlor/reasoner.py`**: External HTTP client for OpenRouter / OpenAI endpoints when background research is requested.
8. **`src/parlor/modes.py`**: Immutable dataclass definitions for system operational modes (`conversation`, `translate`, `listen`).

---

## Database

**None**. Parlor does not use a persistent database. Session history, mode state, timers, and active streams are held in volatile memory within `websocket_endpoint` closure scope (`src/parlor/server.py:333-340`). Disconnecting a WebSocket session resets the state.

---

## API

### HTTP Endpoints (`src/parlor/server.py`)
* **`GET /`**: Serves `src/parlor/web/index.html` with model label substitution (`src/parlor/server.py:316`).
* **`GET /static/*`**: Static asset mounting for frontend CSS and JS (`src/parlor/server.py:313`).

### WebSocket Protocol (`ws://localhost:8000/ws`)
* **Client → Server**:
  * `{"type": "speech_chunk", "audio": "<b64_wav>", "seq": 0}`: Streamed speech audio chunk.
  * `{"type": "frame", "image": "<b64_jpeg>"}`: Webcam JPEG frame.
  * `{"type": "flush"}`: Force immediate turn response for held audio.
  * `{"type": "interrupt"}`: Abort ongoing LLM generation and TTS playback.
  * `{"type": "set_mode", "mode": "translate"|"listen"|"conversation"}`: Manual UI mode override.
  * `{"type": "cancel_timer", "id": 1}`: Cancel active countdown timer.
  * `{"type": "ready"}`: Client ready signal following audio playback completion.
* **Server → Client**:
  * `{"type": "transcript", "transcription": "...", "p_complete": 0.95}`: Live transcript of user utterance.
  * `{"type": "text_delta", "text": "..."}`: Incremental text delta of assistant reply.
  * `{"type": "audio_start", "sample_rate": 24000}`: Signals beginning of audio stream.
  * `{"type": "audio_chunk", "audio": "<b64_pcm16>", "index": 0}`: PCM audio chunk.
  * `{"type": "audio_end", "tts_time": 0.45}`: End of turn audio stream.
  * `{"type": "turn_final", "timings": {...}, "spoke": true}`: Turn execution metadata.
  * `{"type": "mode_changed", "mode": "..."}`: Confirms mode change.
  * `{"type": "timer_started"|"timer_resolved"}`: Timer event notifications.
  * `{"type": "delegation_started"|"delegation_resolved"|"delegation_parked"}`: Research event notifications.

---

## External Services

1. **HuggingFace Hub (`huggingface_hub`)**: Used at startup to download GGUF models (`google/gemma-4-E4B-it-qat-q4_0-gguf`), multimodal projections, Smart Turn ONNX models, and Kokoro TTS weights (`src/parlor/llama.py:50`, `src/parlor/turn_detector.py:29`, `src/parlor/tts.py:44`).
2. **OpenRouter / OpenAI Chat API**: Optional HTTP endpoint (`https://openrouter.ai/api/v1`) invoked via `reasoner.py:48` when `REASONER_API_KEY` is present.

---

## Configuration

Environment variables (loaded via `python-dotenv` from `.env`):

| Variable | Default | Purpose | File Reference |
| :--- | :--- | :--- | :--- |
| `MODEL` | `e4b` | Model tier (`e2b`, `e4b`, `12b`) | `src/parlor/llama.py:32` |
| `MODEL_PATH` | `""` | Direct override for GGUF model path | `src/parlor/llama.py:43` |
| `MMPROJ_PATH` | `""` | Direct override for mmproj path | `src/parlor/llama.py:44` |
| `LLAMA_PORT` | `8081` | Port for `llama-server` | `src/parlor/llama.py:34` |
| `LLAMA_SERVER_URL` | `""` | External `llama-server` URL | `src/parlor/llama.py:35` |
| `LLAMA_CTX` | `16384` | LLM context window size | `src/parlor/llama.py:36` |
| `TEMPERATURE` | `0.7` | Sampling temperature for speech generation | `src/parlor/llama.py:37` |
| `PORT` | `8000` | FastAPI server port | `src/parlor/server.py:852` |
| `REASONER_API_KEY` | `""` | API key enabling background research | `src/parlor/reasoner.py:19` |
| `REASONER_BASE_URL` | `https://openrouter.ai/api/v1` | Base URL for reasoner API | `src/parlor/reasoner.py:17` |
| `REASONER_MODEL` | `openai/gpt-5.6-luna` | Model identifier for reasoner | `src/parlor/reasoner.py:20` |
| `REASONER_TIMEOUT` | `90` | Request timeout for reasoner (seconds) | `src/parlor/reasoner.py:22` |
| `REASONER_WEB_SEARCH` | `1` | Appends `:online` to model on OpenRouter | `src/parlor/reasoner.py:29` |
| `TIME_NOTE_MIN_S` | `120` | Silence threshold to trigger time notes | `src/parlor/server.py:238` |
| `KOKORO_ONNX` | `""` | Set to force ONNX TTS on Apple Silicon | `src/parlor/tts.py:59` |

---

## Authentication and Security

* **Authentication**: None. Server endpoints (`/`, `/ws`) have no user authentication or token verification.
* **Network Binding**: Defaults to `localhost:8000` (`src/parlor/server.py:856`). The documentation explicitly notes avoiding `0.0.0.0` due to Web API secure context requirements for `getUserMedia`.
* **API Key Handling**: `REASONER_API_KEY` is loaded into environment memory and passed via `Authorization: Bearer` header (`src/parlor/reasoner.py:70`).
* **Input Validation**: Audio messages are validated by checking base64 payload length (`src/parlor/pipeline.py:55`). Images are forwarded without binary header verification.

---

## Testing

* **Framework**: Pytest 8.4 (`pyproject.toml:30`).
* **Execution**: Command `pytest`.
* **Test Architecture**:
  * Unit tests (`tests/test_stream_parser.py`, `tests/test_elapsed.py`, `tests/test_history.py`) run in milliseconds without launching servers.
  * Integration tests (`tests/test_conversation.py`, `tests/test_delegation.py`, `tests/test_timer.py`) use fixtures in `tests/conftest.py` and speech synthesis in `benchmarks/fixtures.py`.
  * Failure & robustness tests (`tests/test_robustness.py`, `tests/test_z_failure.py`) test interruption handling and malformed payloads.

---

## Error Handling

1. **`llama-server` Crashes**: Monitored via health checks (`src/parlor/llama.py:151`). HTTP 400 or non-200 responses from `llama-server` raise `RuntimeError` (`src/parlor/llama.py:231`).
2. **Action Decider Failures**: Handled defensively in `_decide()` (`src/parlor/actions.py:128-139`). Exceptions are logged and return `ActionDecision()`, treating decider errors as no-op turns rather than breaking the turn loop.
3. **Malformed Client Inputs**: Audio segments failing `valid_audio()` are ignored (`src/parlor/server.py:704`). Turn execution errors emit `release_client()` to unblock client state (`src/parlor/server.py:836`).
4. **TTS Generation Fallbacks**: If `mlx-audio` fails to import or load on macOS, `tts.load()` catches `ImportError` and falls back to `ONNXBackend` (`src/parlor/tts.py:64`).

---

## Technical Debt

1. **Global Process State**: `llama.py` uses module-level globals (`_proc`) to manage the `llama-server` subprocess (`src/parlor/llama.py:39`).
2. **Synchronous HTTP in Reasoner**: `reasoner.py` uses blocking `urllib.request.urlopen` (`src/parlor/reasoner.py:72`) executed within a `ThreadPoolExecutor` instead of native `httpx` or `aiohttp` async clients.
3. **Single Session Concurrency**: `llama-server` runs with `-np 1` (`src/parlor/llama.py:145`). If multiple clients connect to `/ws`, context slots and cache states will collide.

---

## Important Findings

1. **Decoupled Action Decider Superiority**: Benchmarking (`benchmarks/archbench.py`) proved that decoupled grammar-forced JSON requests achieved 1.0 recall vs 0.955 for in-band tags, completely eliminating tag leakage into spoken audio (`src/parlor/actions.py:4-14`).
2. **Transcript-First Decoding**: Generating `###TRANSCRIPT:` before assistant responses achieved 0.00 Word Error Rate (WER) vs 0.39 WER when transcribing after responses (`src/parlor/pipeline.py:125-131`).
3. **Acoustic vs LLM Turn Gating**: Using `smart-turn-v3.2` ONNX classifier avoids 2B LLM failure on turn-boundary audio classification (`src/parlor/server.py:48-51`).

---

## Non-Obvious Behavior

1. **Instruction Echo Suppression**: The pipeline inspects transcript deltas for 5-word sequence matches against the prompt instruction (`echoes_instruction()`, `src/parlor/pipeline.py:234`). If matched, the transcript is suppressed from client display to prevent prompt leaks.
2. **Tail Silence Injection**: `pad_tail_silence()` appends 300ms of PCM silence inside the WAV buffer (`src/parlor/pipeline.py:91`). This prevents `llama-server`'s audio encoder from hallucinating word completions at VAD cutoffs.
3. **Double Headroom Context Rotation**: `rotate_history()` drops the oldest 25% of messages when estimated context usage exceeds `LLAMA_CTX - 2 * CONTEXT_HEADROOM` (`src/parlor/server.py:270`, `614`), ensuring message pairs remain aligned on user roles.

---

## Documentation Gaps

1. **Multi-Client Limitations**: Existing docs do not explicitly state that `llama-server` is configured for single-session concurrency (`-np 1`), making multi-tenant deployments unsupported.
2. **ONNX Memory Usage**: No documentation on RAM requirements when loading both `smart-turn-v3.2` and `kokoro-v1.0.onnx` simultaneously on low-memory edge devices.

---

## Uncertainty

* **Observed**: Hardware performance metrics (0.6s–1.7s TTFA on M3 Pro) and test coverage.
* **Inferred**: `max_completion_tokens` handling in `reasoner.py:59` assumes direct compatibility with OpenAI reasoning models if `BASE_URL` contains `api.openai.com`.
* **Unknown**: Maximum WebSocket connection lifespan before network proxies drop idle connections without ping frame configuration.

---

## Questions for Further Investigation

1. Could `llama-server` parallel slots (`-np N`) be enabled without degrading prefix cache performance for single-user voice turns?
2. Would replacing `urllib.request` in `reasoner.py` with `httpx.AsyncClient` reduce thread pool overhead during concurrent background searches?
