# ComfyUI Workflow JSON Format (API Format)

Verified against the installed ComfyUI 0.27.1 (`workspace/nodes.py`, `workspace/execution.py`, `workspace/script_examples/`). For upstream docs see the `comfyui-docs` skill (workflow section).

## Structure

The API format — what `POST /prompt` accepts and what `comfy run` executes — is a single flat JSON object mapping node-id strings to node objects:

```json
{
  "<node_id>": {
    "class_type": "<RegisteredNodeClassName>",
    "inputs": {
      "<literal_input>": <scalar>,
      "<connection_input>": ["<source_node_id>", <output_slot_index>]
    },
    "_meta": { "title": "<display title, optional>" }
  }
}
```

- Node IDs are arbitrary strings (conventionally numeric: `"3"`, `"4"`); they need not be contiguous.
- Literal inputs are scalars (string, number, bool). Connection inputs are **exactly** a length-2 array `[source_node_id_string, output_slot_int]`; any other list shape is rejected (`bad_linked_input`).
- `_meta.title` is optional; it only improves error messages and readability.
- There is no layout data (positions, links array, groups) in API format.

## Validation rules enforced by the server

- At least one node must be an output node (`SaveImage`, `PreviewImage`, ...) or the prompt is rejected: `prompt_no_outputs`.
- Connected output types must match the input's expected type (`return_type_mismatch`); cycles are rejected (`dependency_cycle`).
- Model filename inputs (`ckpt_name`, `lora_name`, ...) must match files present under `workspace/models/`; `LoadImage.image` must name a file already in `workspace/input/` (upload via `POST /upload/image` first).
- `control_after_generate` seen in the UI is a widget setting, **not** an API input — do not include it.

## UI format vs API format

| | UI save format (`File → Save`) | API format (`File → Export (API)`) |
| --- | --- | --- |
| Top level | `nodes[]`, `links[]`, `groups[]`, `version`, ... | flat `{node_id: {class_type, inputs}}` map |
| Layout data | yes (positions, sizes, colors) | none |
| `POST /prompt` | rejected | accepted |
| `comfy run --workflow` | accepted (converted client-side, needs running server) | accepted |

## Verified minimal text-to-image workflow

Structurally matches `workflows/text-to-image.json` (prompt texts below are placeholders). All input names verified against `INPUT_TYPES` in `workspace/nodes.py`.

```json
{
  "3": {
    "class_type": "KSampler",
    "inputs": {
      "seed": 156680208700286, "steps": 20, "cfg": 8,
      "sampler_name": "euler", "scheduler": "normal", "denoise": 1,
      "model": ["4", 0], "positive": ["6", 0], "negative": ["7", 0],
      "latent_image": ["5", 0]
    },
    "_meta": { "title": "KSampler" }
  },
  "4": {
    "class_type": "CheckpointLoaderSimple",
    "inputs": { "ckpt_name": "v1-5-pruned-emaonly-fp16.safetensors" },
    "_meta": { "title": "Load Checkpoint" }
  },
  "5": {
    "class_type": "EmptyLatentImage",
    "inputs": { "width": 512, "height": 512, "batch_size": 1 },
    "_meta": { "title": "Empty Latent Image" }
  },
  "6": {
    "class_type": "CLIPTextEncode",
    "inputs": { "text": "<positive prompt>", "clip": ["4", 1] },
    "_meta": { "title": "CLIP Text Encode (Positive)" }
  },
  "7": {
    "class_type": "CLIPTextEncode",
    "inputs": { "text": "<negative prompt>", "clip": ["4", 1] },
    "_meta": { "title": "CLIP Text Encode (Negative)" }
  },
  "8": {
    "class_type": "VAEDecode",
    "inputs": { "samples": ["3", 0], "vae": ["4", 2] },
    "_meta": { "title": "VAE Decode" }
  },
  "9": {
    "class_type": "SaveImage",
    "inputs": { "filename_prefix": "ComfyUI", "images": ["8", 0] },
    "_meta": { "title": "Save Image" }
  }
}
```

## Node reference (inputs verified in workspace/nodes.py)

Output slots are positional per class — confirm unfamiliar classes via `GET /object_info/{class}`.

| class_type | Purpose | Inputs (name: kind) | Outputs (slot: type) |
| --- | --- | --- | --- |
| `CheckpointLoaderSimple` | Load checkpoint from `models/checkpoints/` | `ckpt_name`: filename | 0: MODEL, 1: CLIP, 2: VAE |
| `CLIPTextEncode` | Encode prompt text | `text`: string, `clip`: conn CLIP | 0: CONDITIONING |
| `EmptyLatentImage` | Blank latent canvas (txt2img start) | `width`, `height`, `batch_size`: int | 0: LATENT |
| `KSampler` | Denoising sampler | `model`: conn MODEL, `positive`/`negative`: conn CONDITIONING, `latent_image`: conn LATENT, `seed`: int (max 2^64-1), `steps`: int, `cfg`: float, `sampler_name`: e.g. `euler`, `scheduler`: e.g. `normal`, `denoise`: float 0-1 (1.0 = txt2img, <1.0 = img2img strength) | 0: LATENT |
| `VAEDecode` | Latent → image | `samples`: conn LATENT, `vae`: conn VAE | 0: IMAGE |
| `VAEEncode` | Image → latent (img2img entry) | `pixels`: conn IMAGE, `vae`: conn VAE | 0: LATENT |
| `SaveImage` | Persist to `workspace/output/` | `images`: conn IMAGE, `filename_prefix`: string | 0: IMAGE (passthrough; output node) |
| `PreviewImage` | Temp preview (not persisted to output/) | `images`: conn IMAGE | 0: IMAGE (passthrough; output node) |
| `LoraLoader` | Apply LoRA from `models/loras/` (chainable) | `model`: conn MODEL, `clip`: conn CLIP, `lora_name`: filename, `strength_model` / `strength_clip`: float (schema range -100..100; typical 0-1) | 0: MODEL, 1: CLIP |
| `LoadImage` | Read from `workspace/input/` | `image`: filename (must exist server-side) | 0: IMAGE, 1: MASK (inverted alpha) |
| `ImageScale` | Resize without model | `image`: conn IMAGE, `upscale_method`: nearest-exact/bilinear/area/bicubic/lanczos, `width`/`height`: int (0 = keep aspect), `crop`: disabled/center | 0: IMAGE |
| `UpscaleModelLoader` | Load from `models/upscale_models/` | `model_name`: filename | 0: UPSCALE_MODEL |
| `ImageUpscaleWithModel` | Model-based upscale | `upscale_model`: conn UPSCALE_MODEL, `image`: conn IMAGE | 0: IMAGE |

## Recipes

- **img2img**: `LoadImage → VAEEncode → KSampler.latent_image`, set `denoise` ≈ 0.5-0.8. Upload the source image to `workspace/input/` first (`POST /upload/image` or file copy).
- **LoRA**: insert `LoraLoader` after `CheckpointLoaderSimple`; rewire KSampler `model` to LoRA slot 0 and both `CLIPTextEncode.clip` to LoRA slot 1. Chain multiple LoraLoaders for multiple LoRAs.
- **Upscale**: `VAEDecode → ImageUpscaleWithModel (with UpscaleModelLoader) → SaveImage`.
