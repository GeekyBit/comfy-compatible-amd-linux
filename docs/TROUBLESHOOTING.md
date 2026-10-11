# Troubleshooting RDNA2 ComfyUI

Use this order: verify the host, verify ROCm, verify PyTorch, verify the selected GPU, verify the ComfyUI API, then debug the workflow.

## Collect a diagnostic bundle

Run these commands from the ComfyUI workspace. After the repository installer, `comfy -status` is the normal diagnostic command; the explicit CLI path below is used only where direct CLI output is useful.

```bash
export COMFY_CLI="$HOME/.venvs/comfy-cli/bin/comfy"
```

```bash
uname -a
python --version
"$COMFY_CLI" --version
rocminfo | grep -E 'Name:|Marketing Name:|gfx'

"$COMFYUI_HOME/.venv/bin/python" -c '
import torch
print("torch", torch.__version__)
print("hip", torch.version.hip)
print("available", torch.cuda.is_available())
print("count", torch.cuda.device_count())
for i in range(torch.cuda.device_count()):
    print(i, torch.cuda.get_device_name(i))
'

curl -fsS http://127.0.0.1:8188/system_stats
curl -fsS http://127.0.0.1:8188/queue
comfy -status
```

Remove usernames, hostnames, IP addresses, private paths, prompts, and tokens before sharing diagnostics.

## ComfyUI will not start

Check for an existing listener:

```bash
ss -ltnp | grep ':8188'
```

Check the CLI-managed instance:

```bash
comfy -status
```

Stop and restart only the ComfyUI instance managed by this project:

```bash
comfy -stop
comfy -start
```

If the port is occupied by an unrelated process, do not kill it blindly. Identify the owner first.

## `torch.cuda.is_available()` is false

Check:

```bash
ls -l /dev/kfd /dev/dri/render*
id
rocminfo
```

Typical causes:

- Missing or mismatched ROCm user-space libraries.
- The account is not in the `render` or `video` group.
- The kernel KFD device is unavailable.
- A PyTorch wheel for the wrong backend was installed.
- The process is inside a container without `/dev/kfd` and render devices passed through.

Fix the host/runtime problem before changing ComfyUI nodes.

## The installer stops because ROCm-enabled PyTorch is missing

The installer checks the host prerequisites before installing the Comfy CLI or ComfyUI. It checks Bash, Python with venv support, Git, curl, ROCm/`rocminfo`, `/dev/kfd`, and a render device. If one of those is missing, it lists the missing requirement, gives distribution-specific guidance, and exits without installing system packages or application software.

The PyTorch check is different. `comfy-cli` must first create the ComfyUI workspace before the installer can inspect that workspace's Python environment. After the workspace is created, the installer checks that:

```python
import torch
torch.cuda.is_available()
torch.version.hip
```

If the check fails, the installer does not install PyTorch automatically. It prints a suggested command and exits. At that point, the ComfyUI workspace exists, but the ROCm-enabled PyTorch environment still needs to be corrected manually.

First inspect the environment:

```bash
export COMFYUI_HOME="$HOME/ComfyUI"
"$COMFYUI_HOME/.venv/bin/python" -c '
import torch
print("torch:", torch.__version__)
print("hip:", torch.version.hip)
print("available:", torch.cuda.is_available())
print("device_count:", torch.cuda.device_count())
for i in range(torch.cuda.device_count()):
    print(i, torch.cuda.get_device_name(i))
'
```

If the workspace uses `venv` instead of `.venv`, adjust the path accordingly. Install a ROCm-compatible PyTorch wheel only after checking the ROCm version and supported Python version for the target system. The form used by the reference ROCm 7.2 environment is:

```bash
"$COMFYUI_HOME/.venv/bin/python" -m pip install \
  torch torchvision torchaudio \
  --index-url https://download.pytorch.org/whl/rocm7.2
```

Do not copy that command unchanged to a different ROCm release. Confirm the correct wheel index and package versions first, then rerun the verification command. When the check succeeds, run `comfy -setup` to save the launch settings and `comfy -start` to test the server.

Helpful official resources:

- [PyTorch installation selector](https://pytorch.org/get-started/locally/) — choose Linux, Python, pip, and the supported ROCm release.
- [AMD ROCm PyTorch installation guide](https://rocm.docs.amd.com/projects/install-on-linux/en/docs-7.2.3/install/3rd-party/pytorch-install.html) — AMD's ROCm-specific installation guidance.
- [ComfyUI manual installation](https://docs.comfy.org/installation/manual_install) — ComfyUI's Python environment and dependency guidance.
- [`comfy-cli` getting started](https://docs.comfy.org/comfy-cli/getting-started) — official workspace installation, launch, update, and environment commands.

Do not install a CUDA-only PyTorch wheel into this environment. If PyTorch imports but reports no HIP runtime, verify the wheel, ROCm libraries, `/dev/kfd`, render permissions, and the active Python environment before changing workflow nodes.

## The wrong GPU is selected

Print the visibility variables:

```bash
tr '\0' '\n' < /proc/COMFY_PID/environ | grep -E '^(HIP|ROCR)_VISIBLE_DEVICES='
```

Then compare `rocminfo`, PyTorch device names, and ComfyUI `/system_stats`.

Remember that after `HIP_VISIBLE_DEVICES=1`, the selected physical GPU can appear as logical device zero. On the reference V620 system, adding `ROCR_VISIBLE_DEVICES=0` caused selection problems; remove it unless a controlled test proves it is needed.

## Idle ComfyUI uses a full CPU core

First confirm that no workflow is queued:

```bash
curl http://127.0.0.1:8188/queue
```

If the queue is empty, inspect threads:

```bash
ps -eLo pid,tid,pcpu,stat,wchan,comm,args | grep -E 'python|comfy'
top -H -p COMFY_PID
```

On the reference ROCm 7.2/PyTorch stack, test:

```bash
export HSA_TOOLS_DISABLE_REGISTER=1
```

Restart ComfyUI and measure CPU usage, temperature, and power again. This is a runtime workaround observed on one stack, not a universal ComfyUI requirement.

## Port 8188 works locally but not remotely

Check the bind address:

```bash
ss -ltnp | grep ':8188'
```

`127.0.0.1:8188` is local-only. For LAN use, bind to the host's LAN address or an explicitly chosen interface, then allow only the trusted client network through the firewall.

Do not expose unauthenticated ComfyUI directly to the Internet.

## `comfy -status` says the server is not running

The CLI may be querying its default local URL while the server is bound elsewhere. Set the URL explicitly:

```bash
export COMFY_LOCAL_URL=http://HOST_LAN_ADDRESS:8188
comfy -status
```

Or query the API directly:

```bash
curl http://HOST_LAN_ADDRESS:8188/system_stats
```

## Missing node or model errors

Check the model folder and workflow node type. For Z-Image-Turbo, verify:

```text
models/diffusion_models/z_image_turbo_bf16.safetensors
models/text_encoders/qwen_3_4b.safetensors
models/vae/ae.safetensors
```

Install missing dependencies through the ComfyUI manager or the CLI's documented node dependency command. Snapshot first if the environment is known-good.

## VRAM or allocation failures

Reduce one variable at a time:

1. Batch size.
2. Resolution.
3. Model precision or model variant.
4. Concurrent jobs.
5. Optional nodes and previews.

Free cached models through the supported CLI/API operation before restarting the whole server. Confirm that the other GPU workload is not using the same card.

## Generation is slow

Separate these cases:

- First model load versus warm cache.
- Prompt encoding versus diffusion sampling.
- GPU computation versus CPU offload.
- One GPU versus the intended GPU.
- Single image versus batch generation.

Record the workflow, steps, resolution, model files, PyTorch/ROCm versions, and warm/cold state before comparing results.

## Updating broke the installation

Restore the last known-good snapshot or checkout. Do not update ComfyUI, PyTorch, ROCm, and custom nodes simultaneously. Re-run the diagnostic bundle and the known-good Z-Image-Turbo workflow before making another change.
