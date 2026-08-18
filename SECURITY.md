# Security & Threat Model Analysis — Parlor

This document provides a security assessment of **Parlor**, analyzing authentication controls, network exposure, input validation, external communication risks, secret management, and potential attack vectors.

---

## 1. Authentication and Access Control

### 1.1. WebSocket Endpoint Security (`/ws`)
* **Observation**: The main application WebSocket endpoint (`ws://localhost:8000/ws`) contains **no authentication or authorization mechanisms** (`src/parlor/server.py:332`).
* **Impact**: Any process or script capable of opening a socket connection to `localhost:8000` can initiate WebSocket sessions, capture model output, interact with system mode state, and trigger background research calls.
* **Mitigation / Risk Control**: The application is explicitly designed for **single-user local desktop operation**. `server.py` binds to `localhost` by default (`src/parlor/server.py:856`).

---

## 2. Network Exposure & Interface Binding

### 2.1. Localhost Host Binding
* **Implementation**: `uvicorn.run(app, host="localhost", port=port)` (`src/parlor/server.py:856`).
* **Design Rationale**: Binding to `localhost` (rather than `0.0.0.0`) ensures that modern web browsers treat the application as a **Secure Context** (`http://localhost`). Browsers strictly disallow access to media capture APIs (`navigator.mediaDevices.getUserMedia`) when served over non-secure HTTP hostnames like `http://0.0.0.0`.
* **Security Risk**: If `PORT` is bound to a public interface (`0.0.0.0`) in deployment environments, unauthenticated users could access the microphone/camera application interface.

---

## 3. Secrets Management & API Keys

### 3.1. Environment Variable Loading
* **Implementation**: `python-dotenv` loads environment configuration from local `.env` files (`src/parlor/server.py:40`).
* **Key Exposed**: `REASONER_API_KEY` (`src/parlor/reasoner.py:19`).
* **Security Controls**:
  * `.gitignore` explicitly includes `.env` to prevent committing secrets to source repositories (`.gitignore`).
  * `reasoner.py` injects `REASONER_API_KEY` into HTTP `Authorization: Bearer <key>` headers using Python's standard `urllib.request` (`src/parlor/reasoner.py:70`).
  * Log statements explicitly omit raw API key contents.

---

## 4. Input Validation & Data Sanitization

### 4.1. Audio Payload Validation
* **Function**: `valid_audio(b64: str | None) -> bool` (`src/parlor/pipeline.py:53`).
* **Validation Rule**: Verifies that the decoded base64 byte array exceeds `44 + 3200` bytes (~100ms of 16kHz 16-bit PCM WAV audio).
* **Vulnerability Assessment**: `valid_audio()` validates size but **does not perform binary header inspection** prior to standard library parsing. Malformed WAV files that pass length checks could raise unhandled exceptions in `wave.open()` (`src/parlor/pipeline.py:86`).
* **Resilience**: `server.py` wraps turn execution in generic `except Exception` blocks, calling `release_client(ws)` to prevent session crashes on corrupted audio inputs (`src/parlor/server.py:835`).

### 4.2. Camera Frame Image Processing
* **Observation**: Base64-encoded JPEG image payloads (`type: "frame"`) received via WebSocket (`src/parlor/server.py:694`) are forwarded directly to `user_content()` and formatted into `image_url` dicts (`src/parlor/pipeline.py:40`) without server-side resolution caps or MIME type verification.
* **Risk**: Excessive frame payload sizes could induce memory inflation in `llama-server`.

---

## 5. External Network Communication & SSRF Risks

### 5.1. Reasoner Base URL Configuration
* **Implementation**: `BASE_URL = os.environ.get("REASONER_BASE_URL", "https://openrouter.ai/api/v1")` (`src/parlor/reasoner.py:17`).
* **SSRF Risk Assessment**: If `REASONER_BASE_URL` is set to an arbitrary internal IP or host (e.g. `http://169.254.169.254/latest/meta-data/`), background research requests triggered by the LLM (`spawn_delegation`) will POST JSON payloads containing `Authorization: Bearer <REASONER_API_KEY>` to that internal target.
* **Suggested Guard**: Validate `REASONER_BASE_URL` against permitted domain schemas or HTTPS protocols.

---

## 6. Command Execution & Subprocess Isolation

### 6.1. Subprocess Invocation (`llama-server`)
* **Implementation**: `llama.start()` spawns `llama-server` using `subprocess.Popen` (`src/parlor/llama.py:144`).
* **Command Construction**: `server_command()` resolves binary paths using `shutil.which("llama-server")` or `shutil.which("llama")` (`src/parlor/llama.py:102`).
* **Security Evaluation**: Arguments are passed as an explicit list `[binary, "-m", model, "--mmproj", ...]` without `shell=True`, preventing shell injection vulnerabilities.

---

## 7. Security Assumptions & Recommendations Matrix

| Security Area | Assumption | Risk Level | Recommended Improvement |
| :--- | :--- | :--- | :--- |
| **Authentication** | Local desktop single-user scenario. | Low (on local host) | Add optional API token authentication for WebSocket connections. |
| **Network Interface** | Application bound exclusively to `localhost`. | Medium (if exposed) | Explicitly enforce local loopback verification in `server.py`. |
| **Input Validation** | Client VAD sends valid WAV base64 streams. | Low | Add WAV header magic-byte validation (`RIFF...WAVE`) in `valid_audio()`. |
| **Reasoner Endpoint** | `REASONER_BASE_URL` points to trusted HTTPS endpoint. | Medium | Restrict `REASONER_BASE_URL` scheme to HTTPS. |
