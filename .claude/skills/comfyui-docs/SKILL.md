---
name: comfyui-docs
description: Look up official ComfyUI documentation at docs.comfy.org (Japanese). Use when node parameters, tutorial workflows, server or API behavior, comfy-cli usage, or troubleshooting guidance is needed beyond what the local skill references cover.
---

# ComfyUI Official Docs Lookup

docs.comfy.org is the authoritative, frequently-updated source. Local references in the other `comfyui-*` skills cover stable knowledge; fetch these pages for anything that changes (node catalog, tutorials, new models, Manager UI).

## Access patterns

- **Japanese pages**: insert `/ja` after the domain — `https://docs.comfy.org/<path>` → `https://docs.comfy.org/ja/<path>`. Nearly the whole site is translated.
- **Raw Markdown**: append `.md` to any page URL (e.g. `https://docs.comfy.org/ja/tutorials/basic/text-to-image.md`) — cleaner for WebFetch than the HTML page.
- **Built-in node pages** are flat at `https://docs.comfy.org/ja/built-in-nodes/<NodeName>` (878 pages). URL casing can differ from `class_type` (e.g. `ClipTextEncode`, not `CLIPTextEncode`).
- **English fallback**: the REST reference under `/api-reference/*` serves English content even at `/ja` URLs; `/changelog` is translated at `/ja/changelog`.
- **Discovery**: `https://docs.comfy.org/llms.txt` is a full English page index (no `/ja` variant); `https://docs.comfy.org/sitemap.xml` is authoritative for all locales.

## When a fetch 404s

1. Retry without `/ja` (page may be English-only or newly restructured).
2. Search `https://docs.comfy.org/llms.txt` for the topic to find the current path, then re-add `/ja`. Note WebFetch truncates this large index — if the topic is not found there (especially built-in-node pages), grep `https://docs.comfy.org/sitemap.xml` instead; it is authoritative and complete.
3. If the page moved permanently, update the URL map below in this skill.

## URL map (curated 2026-07)

### Getting started

| Page | Use when |
| --- | --- |
| <https://docs.comfy.org/ja/get_started/first_generation> | First image generation: loading a workflow, installing the model, running the queue |
| <https://docs.comfy.org/ja/installation/update_comfyui> | Upgrading ComfyUI / version-mismatch issues |
| <https://docs.comfy.org/ja/interface/overview> | Explaining UI areas to a human user |
| <https://docs.comfy.org/ja/interface/features/template> | Built-in workflow templates to start from |

### Workflow format and concepts

| Page | Use when |
| --- | --- |
| <https://docs.comfy.org/ja/development/api-development/workflow-api-format> | UI→API format conversion for `POST /prompt` — key page for headless execution |
| <https://docs.comfy.org/ja/specs/workflow_json> | Authoritative schema of the UI-save workflow JSON |
| <https://docs.comfy.org/ja/specs/nodedef_json> | Interpreting `/object_info` node definitions |
| <https://docs.comfy.org/ja/development/core-concepts/workflow> | Workflow concepts: graph, saving/loading, PNG-embedded metadata |
| <https://docs.comfy.org/ja/development/core-concepts/nodes> | Node anatomy: inputs, widgets, outputs, bypass/mute |
| <https://docs.comfy.org/ja/development/core-concepts/links> | Link semantics and type compatibility |
| <https://docs.comfy.org/ja/development/core-concepts/dependencies> | What external assets a workflow depends on |

### Built-in nodes

| Page | Use when |
| --- | --- |
| <https://docs.comfy.org/ja/built-in-nodes/overview> | Index of all built-in node pages |
| `https://docs.comfy.org/ja/built-in-nodes/<NodeName>` | Any specific node's parameters (e.g. `KSampler`, `CheckpointLoaderSimple`, `ClipTextEncode`, `LoraLoader`, `ControlNetLoader`) |

### Tutorials (workflow recipes)

| Page | Use when |
| --- | --- |
| <https://docs.comfy.org/ja/tutorials/basic/text-to-image> | Canonical SD1.5 txt2img workflow + model placement |
| <https://docs.comfy.org/ja/tutorials/basic/image-to-image> | img2img (VAEEncode + denoise) |
| <https://docs.comfy.org/ja/tutorials/basic/inpaint> / <https://docs.comfy.org/ja/tutorials/basic/outpaint> | Inpainting / outpainting |
| <https://docs.comfy.org/ja/tutorials/basic/upscale> | Model-based upscaling |
| <https://docs.comfy.org/ja/tutorials/basic/lora> / <https://docs.comfy.org/ja/tutorials/basic/multiple-loras> | LoRA usage and chaining |
| <https://docs.comfy.org/ja/tutorials/flux/flux-1-text-to-image> | Flux.1 t2i: required files and Flux-specific graph |
| <https://docs.comfy.org/ja/tutorials/controlnet/controlnet> | ControlNet-guided generation |
| <https://docs.comfy.org/ja/tutorials/image/qwen/qwen-image> | Qwen-Image (strong CJK text rendering) |
| <https://docs.comfy.org/ja/tutorials/video/wan/wan2_2> | Wan2.2 video generation |

### Models

| Page | Use when |
| --- | --- |
| <https://docs.comfy.org/ja/development/core-concepts/models> | Model file types, `models/` layout, `extra_model_paths.yaml` |
| <https://docs.comfy.org/ja/troubleshooting/model-issues> | Models missing from dropdowns, load failures, architecture mismatch |

### Custom nodes and ComfyUI-Manager

| Page | Use when |
| --- | --- |
| <https://docs.comfy.org/ja/installation/install_custom_node> | All install methods (Manager, git clone, comfy-cli) |
| <https://docs.comfy.org/ja/manager/overview> / <https://docs.comfy.org/ja/manager/pack-management> | Manager features; installing/updating packs, missing-node resolution |
| <https://docs.comfy.org/ja/manager/configuration> | Manager security level / network mode (config.ini) |
| <https://docs.comfy.org/ja/manager/troubleshooting> | Manager install errors, security-level blocks |

### comfy-cli

| Page | Use when |
| --- | --- |
| <https://docs.comfy.org/ja/comfy-cli/getting-started> | Core commands overview |
| <https://docs.comfy.org/ja/comfy-cli/reference> | Full command/flag reference |
| <https://docs.comfy.org/ja/comfy-cli/troubleshooting> | comfy-cli failures |
| <https://docs.comfy.org/ja/agent-tools/cli> | Agent-oriented headless workflow execution |

### Server API / development

| Page | Use when |
| --- | --- |
| <https://docs.comfy.org/ja/development/comfyui-server/comms_overview> | Client-server architecture: HTTP + WebSocket flow |
| <https://docs.comfy.org/ja/development/comfyui-server/comms_routes> | HTTP endpoint list |
| <https://docs.comfy.org/ja/development/comfyui-server/comms_messages> | WebSocket message types |
| <https://docs.comfy.org/ja/development/comfyui-server/api-examples> | Working code examples (submit prompt, retrieve images) |
| <https://docs.comfy.org/ja/development/comfyui-server/startup-flags> | Launch flags (`--listen`, `--port`, `--lowvram`, ...) |
| <https://docs.comfy.org/ja/development/comfyui-server/api-key-integration> | Comfy account API key for API Nodes |

### Troubleshooting

| Page | Use when |
| --- | --- |
| <https://docs.comfy.org/ja/troubleshooting/overview> | Start here for any error: diagnosis flow, log locations |
| <https://docs.comfy.org/ja/troubleshooting/custom-node-issues> | Broken startup / missing nodes after installs (bisect strategy) |
| <https://docs.comfy.org/ja/troubleshooting/model-issues> | Model-related failures |

## Precedence

Local ground truth (`workspace/server.py`, `workspace/nodes.py`, `workspace/openapi.yaml`, `--help` output) beats docs when they disagree — the installed versions (see AGENTS.md Overview / `workspace/comfyui_version.py`) may differ from the latest docs. Note the discrepancy when reporting.
