# ComfyUI HTTP API Reference (local server)

Verified against the installed ComfyUI 0.27.1 (`workspace/server.py`, `workspace/openapi.yaml`, `workspace/script_examples/`). Base URL: `http://127.0.0.1:8188`. Every route is also served under the `/api/` prefix (e.g. `/api/prompt`); the shipped `workspace/openapi.yaml` documents the `/api/*` forms.

## Core execution endpoints

| Method | Path | Purpose |
| --- | --- | --- |
| POST | `/prompt` | Submit an API-format workflow. Body: `{"prompt": <workflow>, "client_id": "<uuid>"}`; optional `prompt_id` (must be canonical lowercase UUID), `front` (bool, jump queue head), `extra_data`. Response: `{"prompt_id", "number", "node_errors"}`; 400 with `error` + `node_errors` on validation failure. |
| GET | `/prompt` | Queue summary: `{"exec_info": {"queue_remaining": n}}` |
| GET | `/history/{prompt_id}` | Result for one prompt. `{}` until finished; then `{<id>: {"prompt", "outputs", "status"}}`. `outputs.<node_id>.images[]` = `{filename, subfolder, type}`; `status` = `{status_str: success\|error, completed: bool, messages: [...]}` |
| GET | `/history` | All recent prompts (query: `max_items`, `offset`); capped ring buffer |
| POST | `/history` | `{"clear": true}` or `{"delete": [prompt_id, ...]}` |
| GET | `/view` | Download a file. Query: `filename` (required), `subfolder`, `type` (`output`/`input`/`temp`, default `output`) |
| GET | `/queue` | `{"queue_running": [...], "queue_pending": [...]}` — items are `[number, prompt_id, prompt, extra_data, outputs_to_execute]` |
| POST | `/queue` | `{"clear": true}` or `{"delete": [prompt_id, ...]}` (pending items) |
| POST | `/interrupt` | Stop execution. Optional body `{"prompt_id": "..."}` targets the running prompt; no body = global interrupt |

## Introspection and assets

| Method | Path | Purpose |
| --- | --- | --- |
| GET | `/object_info` | All node classes: required/optional input names, types, defaults, output types. Large — prefer the single-class form. |
| GET | `/object_info/{class}` | One node class (e.g. `/object_info/KSampler`) |
| GET | `/system_stats` | Liveness + versions: `system` (os, comfyui_version, python/pytorch versions, RAM) and `devices` (name, type, VRAM total/free) |
| GET | `/models` | Model folder types (checkpoints, loras, vae, ...) |
| GET | `/models/{folder}` | Filenames in one model folder (404 for unknown folder) |
| GET | `/embeddings` | Available embedding names |
| POST | `/upload/image` | multipart/form-data `image=<file>`; optional `type` (default `input`), `subfolder`, `overwrite`. Response `{name, subfolder, type}` — use `name` as `LoadImage.image` |
| POST | `/upload/mask` | Like `/upload/image` plus `original_ref` JSON |
| POST | `/free` | `{"unload_models": bool, "free_memory": bool}` — release RAM/VRAM between jobs |
| GET | `/api/jobs` | Job list with filtering/sorting/pagination; `GET /api/jobs/{id}`, `POST /api/jobs/{id}/cancel` (state-agnostic cancel: interrupts running, dequeues pending), `POST /api/jobs/cancel` with `{"job_ids": [...]}` for batch cancel |

## WebSocket (progress tracking)

`ws://127.0.0.1:8188/ws?clientId=<uuid>` — use the same UUID as `client_id` in `POST /prompt`; per-execution messages route to the matching socket, `status` broadcasts to all. Connect **before** submitting so no messages are missed. Text messages are `{"type": ..., "data": {...}}`:

| type | Meaning |
| --- | --- |
| `status` | Queue count changed: `data.status.exec_info.queue_remaining` |
| `execution_start` | Prompt began (`prompt_id`) |
| `execution_cached` | Nodes skipped because outputs were cached (`nodes: [...]`) |
| `executing` | Node starting: `{node, display_node, prompt_id}`. **`node == null` with your `prompt_id` = whole prompt finished** (canonical completion signal; the completion frame carries only `node` and `prompt_id`) |
| `progress` | Step progress within a node: `{value, max, node, prompt_id}` |
| `executed` | Output node produced results: `{node, output: {images: [...]}, prompt_id}` — outputs can be harvested here without waiting for history |
| `execution_success` / `execution_error` / `execution_interrupted` | Terminal states; `execution_error` carries `node_id`, `exception_message`, `traceback` |

Binary frames are latent previews: strip the first 8 bytes (two big-endian uint32: event type, image format 1=JPEG/2=PNG); the rest is image bytes.

For simple polling without WebSocket, `GET /history/{prompt_id}` until non-empty is sufficient. The history → outputs → `/view` download flow is demonstrated in `workspace/script_examples/websockets_api_example.py` (which uses the WS null-node signal for completion).

## Gotchas

- `/prompt` accepts **API format only** — a UI-export JSON (`nodes`/`links` arrays) fails validation. `comfy run` accepts both (converts client-side).
- `/history/{prompt_id}` returns `{}` while the job is queued or running — poll until non-empty, don't treat empty as failure.
- Identical prompts may be served from cache: `execution_cached` lists skipped nodes; outputs still appear in history.
- A client-supplied `prompt_id` must be a canonical lowercase hyphenated UUID, otherwise 400 `invalid_prompt_id`.
- `openapi.yaml` documents `/api/history_v2`, but ComfyUI 0.27.1 does not implement it — do not use.
- First generation after server start is slow (checkpoint loads into memory); subsequent runs are much faster.
