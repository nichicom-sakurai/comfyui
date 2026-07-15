---
name: comfyui-workflow
description: Create or edit ComfyUI workflow JSON (API format) in this repository. Use when the user asks to build, modify, or debug an image-generation workflow (text-to-image, img2img, LoRA, upscale) before running it, or when a workflow fails validation.
---

# ComfyUI Workflow Authoring

Create and edit workflow JSON files that can be executed against the local ComfyUI instance. The executable format is the **API format** (flat node map); see [references/workflow-json-format.md](references/workflow-json-format.md) for the full format specification and a verified node reference.

## Procedure

1. **Check available models first.** A workflow referencing a missing model fails validation. List installed checkpoints with `ls workspace/models/checkpoints/`, or when the server is running, `curl -s "http://127.0.0.1:8188/models/checkpoints"`. If a needed model is missing, use the `comfyui-assets` skill.
2. **Start from an existing workflow.** Copy the closest match from `workflows/` (e.g. `workflows/text-to-image.json`) rather than writing from scratch. Patch only the inputs that need to change (prompt text, seed, resolution, model name).
3. **Follow the API-format rules** in [references/workflow-json-format.md](references/workflow-json-format.md). The most common mistakes, all rejected by the server:
   - Missing output node — every workflow needs at least one `SaveImage` / `PreviewImage` (error: `prompt_no_outputs`)
   - Connection values that are not exactly `["<node_id>", <output_slot>]` (error: `bad_linked_input`)
   - UI-export JSON (top-level `nodes` / `links` arrays) submitted where API format is required
4. **Verify unknown node inputs against ground truth.** With the server running: `curl -s "http://127.0.0.1:8188/object_info/<ClassName>"` returns required/optional inputs, types, and defaults. Without the server: read the node's `INPUT_TYPES` in `workspace/nodes.py` (built-ins) or `workspace/comfy_extras/`. For conceptual docs, use the `comfyui-docs` skill.
5. **Validate before running:**

   ```bash
   .venv/bin/comfy --workspace workspace validate --workflow workflows/<name>.json
   ```

   Validation contacts the server on 127.0.0.1:8188 for node definitions; start it first (`mise run start`) or pass `--input <saved object_info JSON>` for offline mode.
6. **Store the result.** Git-managed, reusable workflows belong in `workflows/<purpose>.json` (API format, pretty-printed, 2-space indent). One-off experiments do not need to be committed. Never save into `workspace/user/` — that is ComfyUI's own save area.

## Patching an existing workflow

API-format workflows are patched by editing node inputs in place — the node IDs are stable string keys:

- Prompt text: the `text` input of the `CLIPTextEncode` node wired to KSampler `positive` (check `_meta.title` to tell positive from negative)
- Seed / steps / cfg / sampler: inputs of the `KSampler` node
- Resolution: `width` / `height` of `EmptyLatentImage`
- Model: `ckpt_name` of `CheckpointLoaderSimple` — must exactly match a filename in `workspace/models/checkpoints/`

## Converting from UI format

If the user provides a UI-export JSON (has `nodes` and `links` arrays) or a PNG with embedded workflow metadata, do not hand-convert it. Ask them to re-export via the ComfyUI menu **File → Export (API)**, or run it directly with `.venv/bin/comfy --workspace workspace run --workflow <file> --wait` which converts UI format client-side (requires a running server).
