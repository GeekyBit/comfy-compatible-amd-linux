# Verified upstream links

The links below were checked from the documentation build environment and returned successful HTTP responses at publication time. Versioned documentation can change; use the exact release documentation when reproducing an older stack.

## Core software

- [ComfyUI repository](https://github.com/Comfy-Org/ComfyUI) — source, releases, issues, and workflows.
- [comfy-cli repository](https://github.com/Comfy-Org/comfy-cli) — official CLI for installing and managing ComfyUI.
- [ComfyUI manual installation](https://docs.comfy.org/installation/manual_install) — virtual environments, dependencies, and Linux installation.
- [comfy-cli getting started](https://docs.comfy.org/comfy-cli/getting-started) — CLI installation, workspace management, launch, and background mode.
- [ComfyUI workflow templates](https://github.com/Comfy-Org/workflow_templates) — official workflow JSON templates.

## ROCm and PyTorch

- [AMD ROCm ComfyUI guide](https://rocm.docs.amd.com/projects/ai-ecosystem/en/latest/inference/comfy.html) — AMD's ComfyUI installation overview.
- [AMD ROCm PyTorch installation](https://rocm.docs.amd.com/projects/install-on-linux/en/docs-7.2.3/install/3rd-party/pytorch-install.html) — ROCm/PyTorch installation guidance.
- [AMD ROCm Radeon system requirements](https://rocm.docs.amd.com/projects/radeon-ryzen/en/docs-7.2.1/docs/shared/hipsdk/reference/system-requirements.html) — GPU and architecture support information.
- [PyTorch local installation selector](https://pytorch.org/get-started/locally/) — select Linux, Python, pip, and the matching ROCm release.

## Z-Image-Turbo

- [Official ComfyUI Z-Image-Turbo tutorial](https://docs.comfy.org/tutorials/image/z-image/z-image-turbo) — workflow and model placement instructions.
- [Comfy-Org Z-Image-Turbo model repository](https://huggingface.co/Comfy-Org/z_image_turbo) — official model files.

## Link policy

Prefer the upstream ComfyUI, AMD ROCm, PyTorch, and Hugging Face pages above. Avoid copying model files or workflows from unverified mirrors when the official source is available. Record the model revision and file checksums for production deployments.
