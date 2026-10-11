# Arch / CachyOS installation

The repository installer is the easiest path for a normal user. It installs a `comfy` wrapper whose flags are documented in the README. The commands below describe the underlying official CLI for people who want to perform the installation manually.

```bash
export COMFY_CLI="$HOME/.venvs/comfy-cli/bin/comfy"
```

This is the procedure used for the reference system. It keeps ComfyUI in a private workspace and uses the official `comfy-cli` manager.

## 1. Verify the GPU and ROCm runtime

Use the ROCm installation already supported by the host distribution. Do not mix packages from an Ubuntu ROCm repository into Arch.

```bash
rocminfo | less
/opt/rocm/bin/rocminfo | less
```

Confirm that the target GPU reports `gfx1030`. Confirm that the kernel exposes `/dev/kfd` and a render node:

```bash
test -e /dev/kfd && echo kfd-present
test -e /dev/dri/renderD128 && echo render-node-present
id
```

The exact render-node number may differ. The account running ComfyUI must have access to the relevant `video` and `render` groups.

## 2. Install basic user-space tools

Use the distribution's packages:

```bash
sudo pacman -Syu --needed python git curl
```

Python 3.10 or newer is required by the current `comfy-cli`; Python 3.12 was used for the reference installation.

## 3. Install and pin `comfy-cli`

```bash
python -m venv "$HOME/.venvs/comfy-cli"
source "$HOME/.venvs/comfy-cli/bin/activate"
python -m pip install --upgrade pip
python -m pip install "comfy-cli==1.22.0"
"$COMFY_CLI" --version
```

If `comfy-cli` is installed with another user-level tool such as `uv`, verify the actual executable and version before continuing.

## 4. Install ComfyUI into a workspace

```bash
export COMFYUI_HOME="$HOME/ComfyUI"
"$COMFY_CLI" --workspace="$COMFYUI_HOME" install
```

Pin the ComfyUI checkout used by the reference guide:

```bash
"$COMFY_CLI" --workspace="$COMFYUI_HOME" update comfy --version 0.39.2
```

The version command checks out the requested release. A detached Git HEAD is expected for a version-pinned checkout.

## 5. Install ROCm-enabled PyTorch

If `comfy-cli` selected the correct ROCm environment automatically, verify it first:

```bash
"$COMFYUI_HOME/.venv/bin/python" -c '
import torch
print(torch.__version__)
print(torch.version.hip)
print(torch.cuda.is_available())
print(torch.cuda.device_count())
for i in range(torch.cuda.device_count()):
    print(i, torch.cuda.get_device_name(i))
'
```

If PyTorch must be installed manually, use the official ROCm/PyTorch instructions for the exact ROCm release and Python version. For a ROCm 7.2 environment, the official PyTorch index is the starting point:

```bash
"$COMFYUI_HOME/.venv/bin/python" -m pip install \
  torch torchvision torchaudio \
  --index-url https://download.pytorch.org/whl/rocm7.2
```

Do not run that command over an already validated environment without recording the current packages first. The reference stack was tested with PyTorch `2.14.1+rocm7.2`; changing the wheel set may change behavior.

## 6. GPU selection

Single-GPU host:

```bash
export HIP_VISIBLE_DEVICES=0
```

Dual-GPU host, ComfyUI on the second physical GPU:

```bash
export HIP_VISIBLE_DEVICES=1
```

The visible device is renumbered inside the process. Therefore, PyTorch may report the selected physical card as logical `cuda:0`; that is normal. Verify the card name rather than relying only on the logical index.

On the reference V620 system, do not combine `HIP_VISIBLE_DEVICES=1` with `ROCR_VISIBLE_DEVICES=0`. That combination hid or misselected the intended HIP device. Use the tested HIP variable first; add other visibility variables only when independently verified.

## 7. Optional idle-CPU workaround

The reference ROCm/PyTorch stack could busy-spin an asynchronous-event thread while ComfyUI was idle. The workaround was:

```bash
export HSA_TOOLS_DISABLE_REGISTER=1
```

This is a runtime workaround, not a replacement for ROCm or PyTorch. Apply it when the idle CPU symptom is reproduced; document whether it is needed on the target system.

## 8. Start and verify

```bash
export HIP_VISIBLE_DEVICES=0
export HSA_TOOLS_DISABLE_REGISTER=1
"$COMFY_CLI" --workspace="$COMFYUI_HOME" launch --background -- \
  --listen 127.0.0.1 --port 8188
```

```bash
curl -fsS http://127.0.0.1:8188/system_stats
curl -fsS http://127.0.0.1:8188/queue
```

For remote LAN use, replace `127.0.0.1` with the host's LAN address and apply a restrictive firewall rule. See the security notes in the README and troubleshooting guide.
