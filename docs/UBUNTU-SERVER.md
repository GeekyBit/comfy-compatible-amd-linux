# Ubuntu Server installation

The repository installer is the easiest path for a normal user. It installs a `comfy` wrapper whose flags are documented in the README. The commands below describe the underlying official CLI for people who want to perform the installation manually.

```bash
export COMFY_CLI="$HOME/.venvs/comfy-cli/bin/comfy"
```

This procedure adapts the tested Arch/CachyOS setup to Ubuntu Server. The exact Ubuntu release, kernel, ROCm release, and GPU support matrix must be checked together before installation.

## 1. Choose a supported base

Use an Ubuntu release listed by the AMD ROCm documentation for the ROCm version you intend to run. Do not assume that the newest Ubuntu release and the newest ROCm release are interchangeable.

The reference system used ROCm 7.2.4. On Ubuntu, install ROCm using AMD's official Linux instructions for the selected Ubuntu release, then verify the resulting runtime before installing Python packages.

```bash
rocminfo | less
/opt/rocm/bin/rocminfo | less
```

The output must identify the target GPU as `gfx1030`.

## 2. Install host prerequisites

For Ubuntu 24.04 with Python 3.12 available from the distribution:

```bash
sudo apt update
sudo apt install -y git curl python3 python3-venv python3-pip
```

If the chosen Ubuntu release does not provide the required Python version, install the version supported by the ComfyUI/ROCm wheel combination. Do not mix a Python wheel built for another interpreter ABI.

Verify GPU device permissions:

```bash
ls -l /dev/kfd /dev/dri/render*
id
```

The service account must be in the groups that can access `/dev/kfd` and the render node, commonly `video` and `render`:

```bash
sudo usermod -aG video,render "$USER"
```

Log out and back in after changing group membership.

## 3. Install the official CLI and ComfyUI

```bash
python3 -m venv "$HOME/.venvs/comfy-cli"
source "$HOME/.venvs/comfy-cli/bin/activate"
python -m pip install --upgrade pip
python -m pip install "comfy-cli==1.22.0"

export COMFYUI_HOME="$HOME/ComfyUI"
"$COMFY_CLI" --workspace="$COMFYUI_HOME" install
"$COMFY_CLI" --workspace="$COMFYUI_HOME" update comfy --version 0.39.2
```

`comfy-cli` creates and manages the ComfyUI workspace environment. Keep this environment separate from other ROCm applications.

## 4. Install or verify PyTorch for ROCm

The AMD ROCm documentation and the PyTorch installation selector are authoritative for the exact wheel command. For a ROCm 7.2 wheel index, the form is:

```bash
"$COMFYUI_HOME/.venv/bin/python" -m pip install \
  torch torchvision torchaudio \
  --index-url https://download.pytorch.org/whl/rocm7.2
```

Only run this when creating a new environment or when intentionally replacing the existing PyTorch stack. For a working installation, first record the current package set:

```bash
"$COMFYUI_HOME/.venv/bin/python" -m pip freeze > comfyui-freeze-before-change.txt
```

Verify:

```bash
"$COMFYUI_HOME/.venv/bin/python" -c '
import torch
print("torch", torch.__version__)
print("hip", torch.version.hip)
print("available", torch.cuda.is_available())
for i in range(torch.cuda.device_count()):
    print(i, torch.cuda.get_device_name(i))
'
```

## 5. Launch on the desired V620

```bash
export HIP_VISIBLE_DEVICES=0
export HSA_TOOLS_DISABLE_REGISTER=1
"$COMFY_CLI" --workspace="$COMFYUI_HOME" launch --background -- \
  --listen 127.0.0.1 --port 8188
```

For a second physical GPU in a dual-GPU server, use the physical index verified with `rocminfo`, commonly:

```bash
export HIP_VISIBLE_DEVICES=1
```

Do not infer the index from a different runtime. Verify the selected card from PyTorch and `/system_stats`.

## 6. Health checks

```bash
curl -fsS http://127.0.0.1:8188/system_stats
curl -fsS http://127.0.0.1:8188/queue
"$COMFY_CLI" env
```

When querying a LAN-bound instance through the CLI, set the CLI's local URL variable as documented by the installed version:

```bash
export COMFY_LOCAL_URL=http://HOST_LAN_ADDRESS:8188
"$COMFY_CLI" system-stats
```

## Ubuntu-specific cautions

- Keep the ROCm user-space libraries, PyTorch wheels, and kernel driver compatible.
- Do not install an Ubuntu ROCm repository package set on Arch, or copy an Arch library tree to Ubuntu.
- Do not use a CUDA-only PyTorch wheel.
- If a kernel update changes KFD/render permissions, re-check `/dev/kfd`, group membership, and `rocminfo` before debugging ComfyUI.
- Pin the working ComfyUI/PyTorch versions before adding custom nodes.
