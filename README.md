# ComfyUI-Krea2-Ostris-Edit

Nodes for running Krea 2 edit LoRAs trained with ai-toolkit (`krea2` arch with
`model_kwargs.edit: true`). 

## Installation

Clone this repo into your ComfyUI `custom_nodes` folder and restart ComfyUI:

```bash
cd ComfyUI/custom_nodes
git clone https://github.com/ostris/ComfyUI-Krea2-Ostris-Edit.git
```

No extra dependencies are required. The nodes appear under the `ostris/krea2`
category.

## Nodes

### Text Encode Krea 2 Ostris Edit

Inputs: `clip`, `prompt`, optional `vae`, `image1`..`image3`, `vlm_max_edge`
(default 384), `ref_max_pixels` (default 1048576).
Output: `CONDITIONING`.

Encodes the prompt together with the reference images through the Krea 2
Qwen3-VL text encoder, using Krea's conditioning template with
`Picture N:` vision placeholders — the same layout used during training.
When a VAE is connected, each reference image is also VAE-encoded and attached
to the conditioning as reference latents for the model patch node.

Image sizing: images fed to the Qwen3-VL encoder are downscaled (never
upscaled) so their longest edge fits `vlm_max_edge` (default 384); reference
latents fit a `ref_max_pixels` pixel budget (default 1MP, 0 disables the
limit).

Note: the text encoder checkpoint must include the Qwen3-VL vision weights or
the images cannot be encoded.

### Krea 2 Ostris Edit Model Patch

Inputs: `model`, `kv_cache` (default off). Output: `model`.

Patches the Krea 2 model so it consumes the reference latents from the
conditioning. Each reference is appended to the image token sequence and
conditioned at timestep 0 (the `index_timestep_zero` reference method), and
the denoising prediction covers only the target image tokens. This node is
required because the stock Krea 2 model in ComfyUI ignores reference latents.

If the conditioning has no reference latents, the patched model behaves
exactly like the stock model, so it is safe to leave in the graph.

`kv_cache` caches the reference tokens' attention K/V: they are precomputed in
a single t=0 pass and reused on every denoising step, so the references never
ride along in the per-step sequence (faster, especially at many steps). **The
LoRA must be trained with ai-toolkit's `kv_cache` model kwarg for this to work
properly** — leave it off for normally trained edit LoRAs.

## Example wiring

```
Load Diffusion Model (krea2) -> Load LoRA -> Krea 2 Ostris Edit Model Patch -> KSampler
CLIPLoader (krea2) -> Text Encode Krea 2 Ostris Edit (prompt + images + VAE) -> positive
CLIPLoader (krea2) -> Text Encode Krea 2 Ostris Edit (negative prompt)      -> negative
```
