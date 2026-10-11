# Portability, hardware targeting, and version policy

## What `gfx1030` means

The reference target is RDNA2 `gfx1030`, but the launcher does not hard-code that architecture. It checks that ROCm and PyTorch can see an AMD device and leaves the actual architecture choice to the installed ROCm stack. Another AMD generation can use the same launcher when its ROCm support, PyTorch wheel, drivers, and workflow requirements are compatible.

## Expected portability

The approach is most directly applicable to:

- Radeon Pro V620 (`gfx1030`).
- Radeon RX 6800/6800 XT/6900 XT class cards that expose `gfx1030` and are supported by the chosen ROCm release.
- Other RDNA2 cards only when their actual reported target and ROCm support match the environment.

Do not infer compatibility from the marketing name. Verify `rocminfo` and the ROCm support matrix for the exact model.

## Hardware variables that matter

- VRAM capacity determines which models and resolutions fit.
- System RAM matters when PyTorch or ComfyUI offloads data.
- PCIe topology can affect transfers and multi-GPU behavior.
- Kernel/KFD permissions determine whether HIP can see the card.
- Cooling and power limits affect sustained generation speed.
- ROCm and PyTorch versions affect kernel coverage and stability.

## Reference software matrix

```text
OS:            CachyOS / Arch-derived Linux
ROCm:          7.2.4
Python:        3.12.15
ComfyUI:       0.39.2
comfy-cli:     1.22.0
PyTorch:       2.14.1+rocm7.2
HIP runtime:   7.2.53211
GPU target:    gfx1030
```

Use the same matrix when reproducing the reference result. For a new system, pin a coherent set rather than copying only one package version.

## Version changes

### Safe order

1. Record the working versions and `pip freeze`.
2. Snapshot custom nodes.
3. Update ComfyUI only.
4. Test the known-good workflow.
5. Update one custom node or dependency.
6. Test again.
7. Change PyTorch/ROCm only when required and with a rollback path.

### Avoid

- Installing a CUDA wheel into a ROCm environment.
- Mixing ROCm major releases in one Python environment.
- Applying Ubuntu ROCm package instructions to Arch.
- Replacing a working PyTorch wheel set during a ComfyUI-only update.
- Assuming a successful import means the intended GPU is selected.

## Single-GPU adaptation

For one `gfx1030` GPU:

```bash
export HIP_VISIBLE_DEVICES=0
```

Do not include the dual-GPU allocation language in the launcher. The rest of the ComfyUI workflow is unchanged.

## Dual-GPU adaptation

For a second physical GPU:

```bash
export HIP_VISIBLE_DEVICES=1
```

Verify the physical identity using device names. Do not rely on logical `cuda:0`/`cuda:1` labels after visibility filtering.

## Public deployment

This guide targets local or trusted-LAN use. ComfyUI is not configured here with an authentication layer. If a deployment needs Internet exposure, put it behind an authenticated reverse proxy or VPN and do not forward the unauthenticated ComfyUI port directly.
