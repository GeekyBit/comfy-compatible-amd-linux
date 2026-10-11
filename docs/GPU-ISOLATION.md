# GPU isolation and launch policy

## Why isolation matters

A dual-V620 host can run independent workloads, but a process must be explicitly assigned to a GPU. ComfyUI image/video generation and an LLM server should not silently select the same card.

The tested policy was:

```text
Physical GPU 0: LLM workload
Physical GPU 1: ComfyUI image/video workload
```

For a single-GPU host, use logical device `0`.

## Environment variables

Primary selector:

```bash
HIP_VISIBLE_DEVICES=0
```

or, on the second physical card:

```bash
HIP_VISIBLE_DEVICES=1
```

The process sees only the selected device, and the visible device is remapped to logical device zero inside many libraries. A PyTorch result such as `cuda:0 AMD Radeon Graphics` can therefore still represent the second physical GPU when `HIP_VISIBLE_DEVICES=1` was used.

## Verification sequence

Run the host-level probe:

```bash
rocminfo | grep -E 'Name:|Marketing Name:|gfx'
```

Run the ComfyUI environment probe:

```bash
"$COMFYUI_HOME/.venv/bin/python" -c '
import torch
print("available:", torch.cuda.is_available())
print("count:", torch.cuda.device_count())
for i in range(torch.cuda.device_count()):
    print(i, torch.cuda.get_device_name(i))
'
```

Run the ComfyUI API probe:

```bash
curl -fsS http://127.0.0.1:8188/system_stats
```

Require all three observations to agree with the intended card.

## Do not rely on this pattern without testing

```bash
HIP_VISIBLE_DEVICES=1 ROCR_VISIBLE_DEVICES=0 ...
```

On the reference V620 system this combination selected or exposed the wrong HIP device. Use the simplest selector that works, normally `HIP_VISIBLE_DEVICES` alone.

## Reusable launcher template

Replace the variables with values appropriate for the target host:

```bash
#!/usr/bin/env bash
set -euo pipefail

COMFYUI_HOME="${COMFYUI_HOME:-$HOME/ComfyUI}"
COMFYUI_HOST="${COMFYUI_HOST:-127.0.0.1}"
COMFYUI_PORT="${COMFYUI_PORT:-8188}"
COMFYUI_GPU="${COMFYUI_GPU:-0}"
COMFY_CLI="${COMFY_CLI:-$HOME/.venvs/comfy-cli/bin/comfy}"

export HIP_VISIBLE_DEVICES="$COMFYUI_GPU"
export HSA_TOOLS_DISABLE_REGISTER="${HSA_TOOLS_DISABLE_REGISTER:-1}"

"$COMFY_CLI" --workspace="$COMFYUI_HOME" stop >/dev/null 2>&1 || true
"$COMFY_CLI" --workspace="$COMFYUI_HOME" launch --background -- \
  --listen "$COMFYUI_HOST" --port "$COMFYUI_PORT"

for _ in $(seq 1 180); do
  if curl -fsS "http://${COMFYUI_HOST}:${COMFYUI_PORT}/system_stats" >/dev/null; then
    echo "ComfyUI is ready"
    exit 0
  fi
  sleep 1
done

echo "ComfyUI did not become healthy" >&2
"$COMFY_CLI" --workspace="$COMFYUI_HOME" logs >&2 || true
exit 1
```

## Idle CPU behavior

On the reference ROCm/PyTorch combination, an idle ComfyUI process could consume approximately one CPU core while no workflow was queued. The observed workaround was:

```bash
HSA_TOOLS_DISABLE_REGISTER=1
```

Measure before and after with `top`, `ps`, and package temperature/power tools. Do not assume this variable is needed on every ROCm version. If it causes a tool-specific regression, remove it and investigate the runtime version instead.

## Coexistence checklist

Before starting ComfyUI alongside another GPU workload:

- Confirm the intended physical GPU and `gfx1030` target.
- Export `HIP_VISIBLE_DEVICES` in the launcher, not only in an interactive shell.
- Verify the server's device through `/system_stats`.
- Keep model VRAM requirements within the selected card.
- Do not use a broad visibility setting that exposes both GPUs unless the workload is intentionally multi-GPU.
