# AutoFigure-Edit Local Setup Notes

Personal notes for running AutoFigure-Edit locally in Docker on a US network. Kept separate from the upstream `README.md` so the file survives `git pull`.

The upstream project ships several defaults that target Chinese users (DNS, providers, API endpoints). This file records the tweaks that make it work outside that network, plus a few bug fixes already applied to this checkout.

## Quick Start

```
docker compose up -d --build
```

Open http://localhost:8000.

If anything fails, work through the checklist below.

## .env Configuration

Copy `.env.example` to `.env`, then fill in the keys you actually have:

```
# HuggingFace token (required for the gated RMBG-2.0 background-removal model)
HF_TOKEN=hf_...

# SAM3 segmentation backend keys (one of these is enough)
ROBOFLOW_API_KEY=...
FAL_KEY=

# DNS override (project default is China DNS; switch to global for US/EU networks)
DOCKER_DNS_1=8.8.8.8
DOCKER_DNS_2=1.1.1.1
```

The DNS override matters even outside China. The project ships `223.5.5.5` and `119.29.29.29` as compose defaults. Those servers return GFW-poisoned answers for `api.openai.com`, `huggingface.co`, and similar domains regardless of where the host machine sits. Setting global DNS makes upstream calls resolve to real IPs.

After editing `.env`, the change takes effect only on container recreation:

```
docker compose up -d
```

`docker compose restart` is not enough; it reuses the previous container's environment.

## LLM Provider Routing (Web UI)

The right panel of the web UI exposes two independent paths:

- **Step 1 Image**: the text-to-image model that produces `figure.png`.
- **Step 4 SVG / Reasoning**: the LLM that produces the SVG template.

| Want to Use | SVG Provider | Image Provider | SVG Model | Image Model | Key Type |
|-------------|--------------|----------------|-----------|-------------|----------|
| Gemini end-to-end | Gemini | Gemini | `gemini-3.1-pro-preview` | `gemini-3.1-flash-image-preview` | Google AI Studio |
| OpenRouter relay | OpenRouter | OpenRouter | `openai/gpt-5` or `google/gemini-3.1-pro-preview` | `google/gemini-3-pro-image-preview` | OpenRouter (`sk-or-v1-...`) |
| OpenAI direct (after DNS fix) | OpenAI Responses | OpenAI Images | `gpt-5.5` | `gpt-image-2` | OpenAI (`sk-...`) |

Notes:

- OpenRouter image models are limited; routing Step 1 through Gemini on OpenRouter is the most reliable option there.
- The Bianxie provider in older versions of this project is now mapped to "Custom" (`api.bianxie.ai/v1`) by the legacy alias.
- OpenAI direct works only when DNS is set to global servers (see above) and there is no upstream firewall or VPN intercepting `api.openai.com`. Switch to OpenRouter if it does not work.

## SAM3 (Step 2: Icon Segmentation)

Default backend is Roboflow. Three options:

1. **Roboflow** (default): free tier ships monthly inference credits that reset on the first of each month. The required key is the **Private API Key** (the masked one starting with `LKky...` in the workspace settings, not the publishable key). Paste it into the SAM3 API KEY field in the web UI, or store it as `ROBOFLOW_API_KEY=...` in `.env`.
2. **fal.ai**: paid per call (~$0.01 to $0.03 per call), with $1 of free credits at signup. Set `FAL_KEY=...` in `.env` and choose `fal.ai API` as the SAM3 backend in the web UI.
3. **Local SAM3**: free but heavy. Requires Python 3.12, PyTorch 2.7+, CUDA 12.6, GPU access, and HuggingFace approval for the SAM3 weights. Worth the setup only for high-volume runs.

To reduce SAM3 cost, shrink the **SAM Prompt** field. The default `icon,person,robot,animal` runs SAM3 four times per figure. Cutting to `icon` runs it once.

## HuggingFace (Step 3: RMBG-2.0)

`briaai/RMBG-2.0` is a gated model. Setup:

1. Visit https://huggingface.co/briaai/RMBG-2.0 and click "Agree and access repository".
2. Make a Read token at https://huggingface.co/settings/tokens.
3. Put the token in `.env` as `HF_TOKEN=hf_...`.

The model downloads on first run and persists in the `hf_cache` Docker volume across container restarts. With the DNS override above, the download takes a few minutes.

## Bug Fixes Already Applied

`autofigure2.py`: the helper `_extract_gemini_image` previously returned `google.genai.types.Image` (a pydantic model with fields `image_bytes`, `mime_type`, `gcs_uri`) directly to the rest of the pipeline, which expected a `PIL.Image`. The downstream call `img.size` then failed with `AttributeError: 'Image' object has no attribute 'size'`. The fix introduces `_coerce_pil_image`, which detects the pydantic case and decodes `image_bytes` through `PIL.Image.open(BytesIO(...))` before returning. The two `as_image()` callsites both go through the helper.

If a `git pull` from upstream brings the bug back, either re-apply the fix in `autofigure2.py` and rebuild the image, or hot-patch the running container with `docker cp autofigure2.py autofigure-edit:/app/autofigure2.py` (a hot patch is reverted by the next `docker compose up -d --build`).

## Output Layout

A successful run writes to `outputs/<job_id>/`:

| File | Stage |
|------|-------|
| `figure.png` | Step 1 raster figure |
| `samed.png` | Step 2 segmentation overlay |
| `boxlib.json` | Step 2 detected boxes |
| `icons/icon_AF*_nobg.png` | Step 3 transparent icon crops |
| `template.svg`, `optimized_template.svg` | Step 4 SVG template (with optional optimization pass) |
| `final.svg` | Step 5 assembled SVG with icons inlined as base64 |

`final.svg` is fully self-contained. Every icon is base64-embedded. The file alone is enough for LaTeX `\includegraphics`, slide insertion, browser preview, or upload.

## Refining the Figure in PowerPoint

`final.svg` from the pipeline is rough. To turn it into an editable PowerPoint deck and pull out the icon assets:

```
python skills/svg-to-pptx/scripts/extract_to_pptx.py todo/final.svg todo/
```

This produces:

- `todo/assets/icons/*.png` (the 22 inlined icons as standalone files, named by SVG id)
- `todo/final_rebuild.pptx` (a single slide rebuilding the figure with PowerPoint shapes)

Bezier paths and arrowheads are skipped; redraw them as PowerPoint connectors. See `skills/svg-to-pptx/SKILL.md` for the full translation table and tuning knobs.

## Common Pitfalls

- **DNS poisoning to Facebook IPs**: `api.openai.com` resolves to a `2a03:2880:...` IPv6 address. The project's China DNS defaults are returning poisoned answers. Set `DOCKER_DNS_1` and `DOCKER_DNS_2` to global servers and recreate the container.
- **`Network is unreachable` errors with IPv6 addresses**: the container has no IPv6 route. The fix is the same DNS override; once the resolver returns IPv4 first, the issue goes away.
- **All UDP 53 timeouts after working setup**: Docker Desktop networking sometimes loses UDP egress. Restart Docker Desktop. TCP connectivity alone is not a workable fallback for Python's stdlib resolver.
- **OpenAI Responses still failing with `Connection refused` after DNS fix**: a corporate firewall or VPN is intercepting `api.openai.com`. Switch to OpenRouter.
- **Hot patch reverted after `docker compose up --build`**: rebuild bakes the local source files into a fresh image. To make a fix permanent, edit the local file before rebuilding rather than only `docker cp`-ing into a running container.

## Container Lifecycle Cheatsheet

| Goal | Command |
|------|---------|
| Start or recreate with current `.env` and `docker-compose.yml` | `docker compose up -d` |
| Rebuild image after editing source files | `docker compose up -d --build` |
| Stop without removing | `docker compose stop` |
| Stream logs | `docker compose logs -f autofigure-edit` |
| Health check | `curl http://localhost:8000/healthz` |
| Hot-patch a file into a running container | `docker cp <local> autofigure-edit:/app/<path>` |
