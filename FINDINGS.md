# Detailed Repository Findings — Parlor

This document outlines architectural, structural, security, and quality findings discovered during the deep code review of **Parlor**.

---

# Finding 1: Single-Tenant Server Architecture Limits Multi-User Concurrency

## Severity
**Medium**

## Location
* [llama.py:145](file:///c:/Users/fadzw/parlor/src/parlor/llama.py#L145)
* [server.py:332](file:///c:/Users/fadzw/parlor/src/parlor/server.py#L332)

## Observation
`llama-server` is spawned with `-np 1` (single slot processing mode), while `websocket_endpoint` maintains shared state and memory cache references without connection locking or multi-session isolation.

## Evidence
```python
# src/parlor/llama.py:145
_proc = subprocess.Popen(
    cmd + ["-m", model, "--mmproj", mmproj, "-ngl", "99",
           "--port", str(PORT), "-c", str(CTX), "-np", "1"],
    stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL,
)
```

## Why It Matters
Opening multiple browser tabs or establishing simultaneous WebSocket connections to `/ws` will cause context slot eviction in `llama-server`. Interleaved request streams will invalidate the single KV cache prefix, degrading performance and potentially leaking conversation history between connections.

## Suggested Direction
If multi-connection support is required, update `llama.py` to allow configurable `--parallel` (`-np N`) slots and implement session-keyed context slot management in `server.py`.

---

# Finding 2: Synchronous Blocking I/O in Reasoner Subsystem

## Severity
**Low**

## Location
* [reasoner.py:72](file:///c:/Users/fadzw/parlor/src/parlor/reasoner.py#L72)
* [server.py:294](file:///c:/Users/fadzw/parlor/src/parlor/server.py#L294)

## Observation
Background web research queries execute synchronous `urllib.request.urlopen` HTTP calls wrapped inside a global `ThreadPoolExecutor` (`REASONER_POOL`).

## Evidence
```python
# src/parlor/reasoner.py:72
with urllib.request.urlopen(req, timeout=TIMEOUT_S) as resp:
    body = json.load(resp)
```

## Why It Matters
Wrapping synchronous socket I/O in worker threads consumes operating system thread resources during 90-second timeouts, introducing potential thread starvation under high delegation volumes.

## Suggested Direction
Refactor `reasoner.py` to use native asynchronous HTTP clients (`httpx.AsyncClient` or `aiohttp`) integrated directly with FastAPI's `asyncio` event loop.

---

# Finding 3: Global Subprocess State Management in Driver Module

## Severity
**Low**

## Location
* [llama.py:39](file:///c:/Users/fadzw/parlor/src/parlor/llama.py#L39)
* [llama.py:134](file:///c:/Users/fadzw/parlor/src/parlor/llama.py#L134)
* [llama.py:167](file:///c:/Users/fadzw/parlor/src/parlor/llama.py#L167)

## Observation
`llama-server` process references are managed via a module-level global variable `_proc`.

## Evidence
```python
# src/parlor/llama.py:39
_proc: subprocess.Popen | None = None

def start() -> None:
    global _proc
    ...
```

## Why It Matters
Global process state prevents instantiating isolated server instances for parallel testing or multi-model routing, complicating unit test isolation.

## Suggested Direction
Encapsulate process management into a dedicated `LlamaServerProcess` class attached to FastAPI's `app.state`.

---

# Finding 4: Heuristic Context Token Estimation Accuracy Variance

## Severity
**Low**

## Location
* [pipeline.py:66-82](file:///c:/Users/fadzw/parlor/src/parlor/pipeline.py#L66-L82)
* [server.py:611-619](file:///c:/Users/fadzw/parlor/src/parlor/server.py#L611-L619)

## Observation
Context window history rotation relies on character division (`len(text) // 4`) and static image weights (`IMAGE_TOKENS = 300`) to estimate token usage before LLM execution.

## Evidence
```python
# src/parlor/pipeline.py:74
if p["type"] == "text":
    total += len(p["text"]) // 4
```

## Why It Matters
Multimodal inputs (such as high-resolution vision frames or non-English speech transcripts) can deviate significantly from 4-character estimates, potentially triggering `rotate_history()` earlier or later than optimal context boundaries.

## Suggested Direction
Rely on exact `prompt_tokens` metadata returned in SSE usage chunks (`ChatStream.prompt_tokens`) as the primary trigger for context window rotation.

---

# Finding 5: Binary Header Omission in Audio Validation

## Severity
**Low**

## Location
* [pipeline.py:53-56](file:///c:/Users/fadzw/parlor/src/parlor/pipeline.py#L53-L56)

## Observation
`valid_audio()` checks base64 payload string length (`len(b64) * 3 // 4 > 44 + 3200`) without verifying WAV binary header signatures (`RIFF...WAVE`).

## Evidence
```python
# src/parlor/pipeline.py:55
def valid_audio(b64: str | None) -> bool:
    return bool(b64) and len(b64) * 3 // 4 > 44 + 3200
```

## Why It Matters
Arbitrary or corrupted non-audio base64 payloads exceeding 3,244 bytes pass validation and trigger runtime exceptions inside `wave.open()` during PCM conversion (`src/parlor/pipeline.py:86`).

## Suggested Direction
Inspect initial base64 bytes for `RIFF` and `WAVE` signatures before attempting audio format decoding.
