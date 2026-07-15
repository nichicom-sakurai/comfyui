---
name: comfyui-run
description: Execute a ComfyUI workflow on the local server and collect the generated images. Use when the user asks to generate an image, run a workflow, check generation progress or queue state, or fetch results.
---

# ComfyUI Workflow Execution

Run workflow JSON files against the local ComfyUI server and report the resulting image paths. Full endpoint details: [references/api-reference.md](references/api-reference.md).

## Preconditions

1. **Server running?** `curl -s http://127.0.0.1:8188/system_stats` — a JSON response means yes.
2. If not: `mise run start`, then poll `system_stats` every 2s until it responds (first responses can take ~10-30s). Inspect startup problems with `.venv/bin/comfy --workspace workspace logs --tail 200`.
3. **Workflow valid?** For new or edited workflows run the validate step of the `comfyui-workflow` skill (`.venv/bin/comfy --workspace workspace validate --workflow <file>`). The referenced checkpoint must exist in `workspace/models/checkpoints/` (`comfyui-assets` skill if missing).

Leave the server running between generations in a session; run `mise run stop` when the user is done.

## Primary path: comfy run

```bash
.venv/bin/comfy --workspace workspace run --workflow workflows/text-to-image.json --wait --timeout 600
```

- **Always pass `--wait`** — in comfy-cli 1.12.0 the default submits and returns immediately without waiting for completion.
- `--timeout` is a per-event silence timeout (seconds), not a wall-clock limit; 600 is safe for first runs where the model loads into memory.
- Accepts both API-format and UI-format JSON (UI format is converted client-side).
- Output is a JSON envelope when not attached to a TTY; on success it reports the generated filenames.

## Alternative path: HTTP API (fine-grained control)

Use when patching inputs per call (seed sweeps, prompt variations) or when `comfy run` reporting is insufficient.

1. Build the request body — the workflow goes under the `prompt` key:

   ```bash
   # payload.json: {"prompt": <API-format workflow object>, "client_id": "<uuid>"}
   curl -s http://127.0.0.1:8188/prompt -X POST -H "Content-Type: application/json" -d @payload.json
   ```

   Write `payload.json` (and any `-o` downloads) to a temporary location outside the repository, or delete them afterwards — they are not gitignored.

   Success: `{"prompt_id": "...", "number": ..., "node_errors": {}}`. A 400 response carries `error` + per-node `node_errors` — fix the workflow, do not retry blindly.
2. Poll for completion (1-2s interval):

   ```bash
   curl -s "http://127.0.0.1:8188/history/<prompt_id>"
   ```

   Returns `{}` while queued/running — poll until the response is non-empty, then read `.<prompt_id>.status.status_str`: `success` or `error` (`completed` is `false` on failures; error details are in `status.messages`).
3. Collect outputs from `.<prompt_id>.outputs.<node_id>.images[]` — each entry is `{filename, subfolder, type}`:

   ```bash
   curl -s "http://127.0.0.1:8188/view?filename=<filename>&subfolder=<subfolder>&type=output" -o <local_path>
   ```

   Files with `type: "output"` also exist directly at `workspace/output/<subfolder>/<filename>` — reading that path is equivalent.

## Queue and interruption

- Queue state: `curl -s http://127.0.0.1:8188/queue` (`queue_running` / `queue_pending`)
- Interrupt current job: `curl -s http://127.0.0.1:8188/interrupt -X POST` (optionally `-d '{"prompt_id": "..."}'` for a targeted interrupt)
- Job listing via CLI: `.venv/bin/comfy --workspace workspace jobs ls`

## Reporting results

Report the absolute path(s) of generated files under `workspace/output/`, plus the seed and model used, so the generation is reproducible. If execution failed, quote `status.messages` (or `comfy run` stderr) rather than guessing at the cause.
