# Reproduction checklist

Use this checklist to reproduce the reference result without carrying over private deployment details.

## Host

- [ ] Linux host with an AMD GPU.
- [ ] Exact GPU model recorded.
- [ ] `rocminfo` reports the expected architecture.
- [ ] `/dev/kfd` and a render node are accessible.
- [ ] Account has the required device-group permissions.
- [ ] ROCm version recorded.
- [ ] Python version recorded.
- [ ] Enough VRAM for the selected workflow.
- [ ] Enough disk space for ComfyUI, model files, and cache.

## Software

- [ ] `comfy-cli` version recorded.
- [ ] ComfyUI version pinned.
- [ ] PyTorch version and `torch.version.hip` recorded.
- [ ] `torch.cuda.is_available()` is true.
- [ ] PyTorch reports the intended card name.
- [ ] No CUDA-only wheel is installed.

## GPU isolation

- [ ] `HIP_VISIBLE_DEVICES` is set in the launcher.
- [ ] The launcher does not silently expose both GPUs.
- [ ] `/system_stats` confirms the intended card.
- [ ] Other workloads are assigned to another GPU or stopped.
- [ ] Idle CPU behavior has been measured.

## Workflow

- [ ] Official Z-Image-Turbo files are in the correct model folders.
- [ ] Official workflow template is available.
- [ ] Workflow imports without missing nodes/models.
- [ ] A 1024×1024, 8-step test completes.
- [ ] First-run and warm-cache timings are recorded separately.
- [ ] Output opens correctly from the normal local output directory.

## Remote access

- [ ] Server bind address is deliberate.
- [ ] Firewall allows only the intended LAN/client range.
- [ ] Authentication or a VPN/reverse proxy is used for untrusted networks.
- [ ] Remote `system_stats`, queue, and one test workflow are verified.

## Change control

- [ ] Current files and versions are backed up.
- [ ] Custom-node snapshot exists.
- [ ] No private paths, IP addresses, credentials, or prompts are in shared logs.
- [ ] A rollback command is documented before updating packages.
