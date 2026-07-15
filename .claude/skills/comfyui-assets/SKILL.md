---
name: comfyui-assets
description: Download models (checkpoints, LoRA, VAE, upscalers) and manage ComfyUI custom nodes. Use when a workflow needs a missing model, the user asks to add or update custom nodes, or installed assets need listing.
---

# ComfyUI Models & Custom Nodes

All commands run from the repository root using the canonical comfy-cli form. Model files land under `workspace/models/`, custom nodes under `workspace/custom_nodes/` (both gitignored).

## Model categories

| Category | Directory (`--relative-path`) | Typical contents |
| --- | --- | --- |
| Checkpoints | `models/checkpoints` | Full SD/SDXL/Flux checkpoint `.safetensors` |
| LoRA | `models/loras` | LoRA weight files |
| VAE | `models/vae` | Standalone VAE files |
| ControlNet | `models/controlnet` | ControlNet / T2I-Adapter models |
| Upscalers | `models/upscale_models` | ESRGAN-class upscale models |
| Embeddings | `models/embeddings` | Textual inversion embeddings |
| CLIP / text encoders | `models/clip`, `models/text_encoders` | Standalone text encoders |
| Diffusion models | `models/diffusion_models`, `models/unet` | UNet-only / diffusion-only weights |

## Downloading a model

```bash
.venv/bin/comfy --workspace workspace model download \
  --url <URL> --relative-path models/<category> --filename <name>.safetensors
```

- **Always pass `--filename`** — without it comfy-cli prompts interactively and blocks non-interactive sessions.
- **Always pass `--relative-path`** — for Civitai URLs comfy-cli can auto-derive `models/<type>/<base>` from API metadata, but for Hugging Face URLs it prompts interactively for the path and blocks non-interactive sessions.
- Baseline text-to-image checkpoint used by `workflows/text-to-image.json` (~2.1 GB, from the official ComfyUI tutorial):

  ```bash
  .venv/bin/comfy --workspace workspace model download \
    --url https://huggingface.co/Comfy-Org/stable-diffusion-v1-5-archive/resolve/main/v1-5-pruned-emaonly-fp16.safetensors \
    --relative-path models/checkpoints --filename v1-5-pruned-emaonly-fp16.safetensors
  ```

- Gated/authenticated downloads: manage tokens with `.venv/bin/comfy auth set` (or `--set-hf-api-token` / `--set-civitai-api-token`). Token commands intentionally require a permission prompt; never write tokens into repository files.
- Model sources: prefer official Comfy-Org mirrors on Hugging Face; tutorial pages on docs.comfy.org state the expected model per workflow (`comfyui-docs` skill).

## Listing and verifying models

```bash
.venv/bin/comfy --workspace workspace model list    # table of downloaded models
ls workspace/models/checkpoints/                            # direct filesystem check
curl -s "http://127.0.0.1:8188/models/checkpoints"          # what the running server sees
```

If a newly downloaded model does not appear in server responses, restart: `mise run stop && mise run start`.

## Custom nodes

Names resolve against the ComfyUI-Manager registry (registry.comfy.org IDs, e.g. `comfyui-impact-pack`).

```bash
.venv/bin/comfy --workspace workspace node install <name>      # install one or more nodes
.venv/bin/comfy --workspace workspace node update <name|all>   # update
.venv/bin/comfy --workspace workspace node show installed      # list (or simple-show installed)
.venv/bin/comfy --workspace workspace node install-deps --workflow=<file.json>  # install nodes a workflow needs
```

- **Restart the server after installing or updating nodes** (`mise run stop && mise run start`) — nodes load at startup.
- Before installing, check the node's registry page or repository (via `comfyui-docs` / web search) — custom nodes execute arbitrary Python.
- Reproducibility snapshot before risky changes: `.venv/bin/comfy --workspace workspace node save-snapshot`.

## Destructive operations — ask first

`model remove`, `node uninstall`, `node disable`, `node restore-snapshot`, and workspace-level `install` / `update` are intentionally not allowlisted and will prompt. Confirm with the user before running them.
