# sd-forge-ipadapter-compat

A small Forge Neo extension that makes IPAdapter Just Work, without sacrificing SageAttention 2 speed for the rest of your generations.

## What it fixes

### 1. `AttributeError: 'dict' object has no attribute 'shape'`

Forge Neo's generic ControlNet `get_control` path is invoked for IPAdapter units too, but IPAdapter passes a **dict** as its conditioning hint (containing `clip_vision`, `image`, `embeds`, etc.) instead of a spatial tensor. The base function then tries `samples.shape` on the dict and crashes.

This extension monkey-patches `ControlNet.get_control` and `T2IAdapter.get_control` to early-return when `cond_hint_original` is a dict. IPAdapter's actual conditioning is applied via UNet patching in `process_before_every_sampling`, so the spatial path skip is harmless.

### 2. `Encountered NaN in Latent` (the IPAdapter black-image problem)

SageAttention 2's INT8 quantization works great for plain SDXL but combines badly with IPAdapter's extra cross-attention layers on FP16-sensitive checkpoints (most notably Illustrious, Pony, NoobAI). The latent values overflow at the end of sampling, producing an all-black output.

This extension auto-detects when an IPAdapter unit is active, transparently swaps `attention_function` from SageAttention to PyTorch SDP for that generation only, then restores Sage afterward. You keep the ~30% speed boost from Sage for normal generations and only pay the speed cost when IPAdapter is actually in use.

## Configuration

None. The extension is fully automatic and always-on. To disable, remove or rename the extension folder.

## Compatibility

- Forge Neo (version `neo-2.22`+ tested)
- The attention swap applies to **any fast attention**: SageAttention, or the Comfy-Kitchen INT8 attention the
  [Jrlabanza build](https://github.com/jrlabanza/jrlabanza-image-generator-core) uses on RTX 50 cards. The console
  prints `[ipadapter-compat] IPAdapter active - swapped ...->PyTorch` and `... restored` around each IPAdapter render
- 2026-11-17 fix: the hooks are installed on the IPAdapter module Forge actually loaded (found in
  `modules.script_loading.loaded_scripts`); before, they went onto a second copy imported from `sys.path` and never ran
- Known issue (Forge Neo, not this extension): `ip-adapter-plus_sdxl_vit-h` gives blown-out, saturated SDXL images
  with any attention backend; the plain `ip-adapter_sdxl_vit-h` works
- Works alongside any existing IPAdapter / ControlNet workflow
- No impact on non-IPAdapter generations — they continue to use whatever attention backend Forge started with

## Why an extension?

The fixes could be inlined into Forge's source, but that:
- Gets clobbered on every Forge update
- Couples the fix to a specific Forge version
- Makes it hard for upstream to track

An extension is self-contained, survives updates, and is easy to remove.
