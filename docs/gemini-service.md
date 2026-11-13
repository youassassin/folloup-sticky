# Gemini Service

This document describes the Gemini integration in the Followup firmware.

`components/gemini_service/` owns:

- Gemini API key storage and precedence
- Gemini authentication state and readiness
- every HTTPS exchange with the Gemini API: authentication, text generation
  (`GenerateText`), and audio transcription (`Transcribe`, including the Files
  API upload and clean-up)
- backend HTTP routes for Gemini settings and runtime state
- Wi-Fi-driven Gemini readiness, consumed by the status bar

It does not own:

- the async lifecycle of a transcription request: that is `transcription_service`,
  which calls `gemini_service::Transcribe`
- summary prompts, chunking, caching, and the request queue: that is
  `summary_service`, which calls `gemini_service::GenerateText`
- archived recording indexing: `recording_archive_service`
- the setup-portal front end: `webserver/` (see `webserver/README.md`), which is
  embedded into `wifi_service` and talks to the routes below

## Ownership

Runtime split:

- `gemini_service`
  - configuration, API key precedence, all Gemini HTTP requests, auth state, and
    backend route handlers
- `transcription_service` / `summary_service`
  - own the worker task or queue for a feature, build the inputs, and publish
    results; they hold no Gemini HTTP code themselves
- `wifi_service`
  - owns the backend HTTP server and hosts the Gemini routes through the existing
    portal route registrar
- `main/app_shell.cpp`
  - initializes the service, forwards Wi-Fi network state into Gemini, logs
    Gemini events, and triggers the Gemini-connected sound cue
- `main/status_bar_runtime.cpp`
  - reflects Gemini-ready state into `epaper_ui::StatusBarState`
- `components/epaper_ui/`
  - renders the Gemini-ready star icon in the status bar
- `feedback_service` / `system_sound_service`
  - play a dedicated Gemini-connected cue when readiness goes from false to true

`app_shell` remains an orchestrator here. The Gemini service owns provider state
and route behavior; `app_shell` only wires events, startup order, and
product-facing reactions.

## Internal Layout

The service is a single implementation file:

- [`components/gemini_service/include/gemini_service.h`](../components/gemini_service/include/gemini_service.h)
- [`components/gemini_service/gemini_service.cpp`](../components/gemini_service/gemini_service.cpp)

Internal responsibilities inside that component:

- NVS read/write of the stored Gemini API key
- sdkconfig fallback key lookup and effective-key precedence
- the `HttpSession` helper and per-request helpers (auth `GET`, `generateContent`,
  resumable file upload, file delete)
- request serialization, retry/backoff, response caps, and key-rejection handling
- auth task lifecycle and stale-result protection
- backend JSON request/response handling

If the file grows further, the natural split is `client` (HTTP and retry),
`worker` (auth task), and `settings_storage` (NVS and precedence).

## API Key Sources

API key precedence:

1. NVS-stored key saved through the backend API
2. built-in `CONFIG_FOLLOWUP_GEMINI_API_KEY`
3. no key configured

Resetting Gemini settings clears only the stored NVS key. It does not clear the
built-in sdkconfig fallback key.

## Build-Time Configuration

The build-time Gemini setting lives under `Folloup Settings`:

- `CONFIG_FOLLOWUP_GEMINI_API_KEY`

This is intended for development and bench testing. A key saved through the
backend API takes precedence over the built-in key. The reproducible default is
set in [`sdkconfig.defaults`](../sdkconfig.defaults) and is empty
(`CONFIG_FOLLOWUP_GEMINI_API_KEY=""`).

## Startup And Readiness Flow

Auth flow:

1. `app_shell` initializes `gemini_service`
2. the service loads the stored NVS key, if present
3. `wifi_service` reaches a connected STA state
4. `app_shell` forwards network state into `gemini_service::SetNetworkState(...)`
5. if a Gemini API key is available and no auth is already in flight, Gemini
   starts authentication automatically
6. Gemini performs a model `GET` against
   `https://generativelanguage.googleapis.com/v1beta/`
7. successful auth marks the runtime snapshot ready
8. the status bar shows the Gemini-ready star icon
9. the feedback layer plays the Gemini-connected cue when readiness goes from
   false to true

Authentication is skipped when:

- no API key is configured
- a request is already in flight
- Wi-Fi is not connected
- Wi-Fi is in access-point mode

`ready` means a request can be issued right now:

- `configured == true`
- `authenticated == true`
- Wi-Fi station connected (as last reported through `SetNetworkState`)

Authentication is kept across a Wi-Fi drop, so `ready` returns on reconnect
without re-validating the key. A change in `ready` caused by connectivity fires a
Gemini event, so the Gemini-connected cue also plays when Wi-Fi comes back.

## Requests

All requests go to `https://generativelanguage.googleapis.com/` with the
`x-goog-api-key` header and TLS validated through the ESP-IDF CRT bundle. The
model is fixed at `models/gemini-2.5-flash-lite`.

| Operation | Calls |
| --- | --- |
| Authentication | `GET v1beta/<model>`. A 2xx marks the key valid and records the model display name. |
| Text generation (`GenerateText`) | `POST v1beta/<model>:generateContent` with a single text part. Used by `summary_service`. |
| Transcription (`Transcribe`) | Three requests: start a resumable upload (`POST /upload/v1beta/files`), stream and finalize the WAV to the returned upload URL, then `generateContent` referencing the uploaded file. The audio goes straight from the PSRAM clip chunks to the socket (header, then PCM), so there is no full copy of the clip in RAM. |
| Clean-up (`DeleteUploadedFile`) | `DELETE v1beta/files/<id>` for the uploaded audio. |

Generation requests send `temperature: 0` and `maxOutputTokens` (512 for
transcription, 2048 for text). The transcription prompt asks for a verbatim
transcript with no commentary or formatting.

### Callers

- `transcription_service` runs `Transcribe` on a short-lived worker task, publishes
  the result, and only then deletes the uploaded file.
- `summary_service` queues requests (depth 4) onto a persistent worker. It builds
  a prompt from the last 3 days of transcripts, estimates its size locally, and
  uses a chunked map-reduce when the estimate exceeds its input budget. Results
  are cached on the SD card.

### Request serialization

Every Gemini HTTPS exchange (authentication, `GenerateText`, `Transcribe`, and the
file delete) runs under one internal mutex. Each exchange opens its own TLS
session, which costs tens of KB of internal RAM, so overlapping transcription,
summary, and auth work is not allowed. A call can therefore block while another
finishes. The lock order is network mutex first, state mutex second.

### Key rejection

If a request made after a successful authentication returns HTTP 401 or 403,
`gemini_service` clears the authenticated state (so `ready` goes false), records
the error, and starts re-authentication. A rejection is ignored if the key
changed while the request was in flight.

### Retry and backoff

Authentication, `GenerateText`, and each step of `Transcribe` retry transient
failures: connection-level `transport_error`, and HTTP 408, 429, 500, 502, 503,
504. There are at most 3 attempts per operation, with a 1 s then 2 s backoff plus
0-499 ms of random jitter. Retries are skipped when Wi-Fi is down, and other
statuses (400, 401, 403, 404, ...) and `response_too_large` are never retried.
Backoff sleeps while holding the network lock, so it delays other Gemini work by a
few seconds at most.

For transcription, the audio upload (start + finalize) is retried as a unit on a
fresh connection, because a failed resumable upload is not resumed. The
`generateContent` step is retried on its own, so a failure there does not re-send
the audio.

### Connection reuse during transcription

The three transcription requests share one TLS connection through `HttpSession`.
This was written against the ESP-IDF v5.4 `esp_http_client` source:

- Every request on the session uses `esp_http_client_open` / `write` /
  `fetch_headers` / `read`, never `esp_http_client_perform`. After the manual flow
  the client is left in its "reading response" state, where `perform()` would not
  send a new request but `open()` does.
- The client's event handler is the only thing that collects the response body
  (and the `x-goog-upload-url` header). It fires in the manual flow too, so the
  read loop only drains the socket; appending from both would duplicate data.
- Request headers are cleared before each request so none leak across.
- ESP-IDF has no keep-alive query. If a reused connection turns out to be stale,
  the session closes it and replays that one request on a fresh connection; the
  replay does not count as a retry.

### Uploaded audio cleanup

`Transcribe` returns the uploaded file's resource name in
`TranscriptionResult::uploaded_file_name`, only if it matches the expected
`files/<id>` shape (it is spliced into a URL). After `transcription_service` has
published the result, it calls `gemini_service::DeleteUploadedFile(...)`. The
delete runs after the result so it adds nothing to the user-visible wait, and it is
best effort: failures are logged and ignored because uploads expire on Google's
side anyway.

### Response and output limits

Response bodies are capped while they are read: 16 KB for auth and the upload
start/finalize calls, 64 KB for `generateContent`. A reply past the cap fails the
request with `response_too_large` instead of growing the heap.

If the model stops at its `maxOutputTokens` ceiling, the service logs a warning
and still returns the (possibly cut-off) text.

### Token budgeting

`summary_service` estimates prompt size locally (about 4 characters per token) to
decide whether to trim or chunk. There is no `countTokens` call.

## Public C++ Snapshot Shapes

Source of truth:

- [`components/gemini_service/include/gemini_service.h`](../components/gemini_service/include/gemini_service.h)

### `gemini_service::SettingsSnapshot`

Fields:

- `configured`
- `has_stored_api_key`
- `has_sdkconfig_api_key`
- `api_key_source`
- `api_key_last4`
- `model_name`

### `gemini_service::RuntimeSnapshot`

Fields:

- `initialized`
- `ready`
- `request_in_flight`
- `auth_checked`
- `authenticated`
- `supports_audio_understanding`
- `supports_structured_output`
- `last_http_status`
- `last_status_message`
- `last_model_resource_name`
- `last_model_display_name`
- `last_error_code`
- `last_error_message`

`supports_audio_understanding` and `supports_structured_output` are hard-coded
`false` and nothing consults them; audio transcription works regardless.
`request_in_flight` covers the authentication task only, not `GenerateText` or
`Transcribe`.

### `gemini_service::Snapshot`

Contains:

- `settings`
- `runtime`

### `gemini_service::SettingsPatch`

Fields:

- `has_api_key`
- `api_key`

### `gemini_service::Result`

Fields:

- `success`
- `validation_error`
- `status_code`
- `field`
- `error_code`
- `message`

### `gemini_service::TextResult` and `TranscriptionResult`

Return values of the synchronous calls (run them from a worker task, never from a
UI or input task). Both carry `success`, `http_status`, `error_code`, and
`error_message`. `TextResult` adds `text`. `TranscriptionResult` adds
`transcript`, `uploaded_file_name`, and timing and size fields
(`clip_duration_ms`, `wav_bytes`, `upload_chunk_count`, `upload_elapsed_ms`,
`total_elapsed_ms`).

## Backend Endpoints

Backend endpoints:

- `GET /api/settings/gemini`
- `PATCH /api/settings/gemini`
- `POST /api/settings/gemini/reset`
- `GET /api/runtime/gemini`

These routes are registered through the existing Wi-Fi backend server. Gemini does
not start its own HTTP server. The setup portal in `webserver/` uses them to save
and clear the API key.

Constraints:

- content type is JSON
- `PATCH` requires a JSON object body
- `PATCH` body must be greater than `0` bytes and at most `512` bytes

## JSON Contracts

### `PATCH /api/settings/gemini`

Request shape:

```json
{
  "api_key": "AIza..."
}
```

Validation rules:

- `api_key` must be present
- `api_key` must be a non-empty string after trimming

### Snapshot Responses

Used by:

- `GET /api/settings/gemini`
- `GET /api/runtime/gemini`
- successful `PATCH /api/settings/gemini`
- successful `POST /api/settings/gemini/reset`

Response shape:

```json
{
  "success": true,
  "message": "Gemini settings loaded",
  "settings": {
    "configured": true,
    "has_stored_api_key": false,
    "has_sdkconfig_api_key": true,
    "api_key_source": "sdkconfig",
    "api_key_last4": "1234",
    "model_name": "models/gemini-2.5-flash-lite"
  },
  "runtime": {
    "initialized": true,
    "ready": true,
    "request_in_flight": false,
    "auth_checked": true,
    "authenticated": true,
    "supports_audio_understanding": false,
    "supports_structured_output": false,
    "last_http_status": 200,
    "last_status_message": "Authenticated with Gemini 2.5 Flash-Lite",
    "last_model_resource_name": "models/gemini-2.5-flash-lite",
    "last_model_display_name": "Gemini 2.5 Flash-Lite",
    "last_error_code": "",
    "last_error_message": ""
  }
}
```

Notes:

- `api_key_last4` is intentionally masked metadata only
- the raw API key is never returned by the backend
- `message` varies by endpoint and outcome

### Error Responses

Error response shape:

```json
{
  "success": false,
  "message": "Gemini API key is required",
  "error_code": "missing_api_key",
  "field": "api_key"
}
```

Common error codes:

- `missing_api_key`
- `invalid_api_key`
- `nvs_write_failed`
- `nvs_clear_failed`
- `not_configured`
- `task_alloc_failed`
- `task_start_failed`
- `response_too_large`
- provider error codes returned from Gemini HTTP error payloads when present
  (for example `RESOURCE_EXHAUSTED`, which the UI surfaces as "Gemini quota
  exceeded")

## UI Integration

Gemini readiness is used in two product-facing ways:

- the status bar shows the star icon when Wi-Fi is connected and Gemini is
  authenticated
- the feedback layer plays a dedicated Gemini-connected cue when readiness goes
  from false to true

The status-bar icon is the star asset already present in `project_assets`.
Transcription and summary results surface through their own pages and toasts (see
`docs/app-architecture.md`).

## Logging

Logs cover:

- service initialization
- auth start, success, and failure
- stale auth result suppression
- retry attempts, key rejection, response-cap hits, and `maxOutputTokens` warnings
- uploaded-file delete results
- backend route registration failures
- NVS read/write/clear failures
- app-facing Gemini event snapshots in `app_shell`

## Not Implemented

- **No queue for failed transcriptions.** If a transcription fails after its
  retries, the recording stays on the SD card without a transcript until the user
  re-runs it from the Details page. `summary_service` skips recordings that have no
  transcript.
- **No cancellation.** An in-flight request runs until it completes or times out.
- **No model selection.** The model is a compile-time constant
  (`kDefaultModelName`).
- **No Gemini call for Vibe Check.** That page only re-runs transcription for idea
  recordings.
