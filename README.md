# comfy-compatible-amd-linux

This project started with a practical question: how can someone run ComfyUI comfortably on an AMD RDNA2 card without having to guess which parts of an NVIDIA-focused guide still apply? The reference machine is built around two Radeon Pro V620 cards, but the useful part of the project is the reasoning behind the setup: how ROCm, PyTorch, ComfyUI, model files, and GPU selection fit together.

The aim is to make a working installation understandable and repeatable. A person should be able to read the introduction, decide where the application and model files belong, follow the path for their Linux distribution, and know what to check when something does not work. The guide separates the exact choices made for the V620 from the choices that can be adapted to another RDNA2 `gfx1030` GPU or, with the appropriate ROCm support, another AMD architecture.

This is not intended to feel like a wall of commands. It is a small project guide with a clear route through the work: understand the setup, install it as a user-run application, choose a model and output arrangement, start it when needed, and keep enough information around to troubleshoot or rebuild it later. The included Bash helper handles the repetitive parts while stopping to ask about decisions that belong to the person operating the machine.

The project is also deliberately honest about its limits. It documents one tested stack rather than promising identical behavior on every AMD card. ROCm support, VRAM, firmware, cooling, Python packages, and ComfyUI changes can all affect the result. Private host details, credentials, prompts, generated media, and private storage arrangements are kept out of the public documentation.

## What is in this repository?

The files are deliberately organized so that a reader can start with the overview and then open only the procedure relevant to their machine:

| File | Purpose |
|---|---|
| `README.md` | Explains the project's goal, reference hardware, tested software stack, architecture, simple setup path, and document map. |
| `setup-comfy.sh` | Interactive Bash installer that creates one user-run ComfyUI command, checks prerequisites without installing missing system packages, configures storage and web access, and optionally creates a user startup service. It does not download model weights. |
| `examples/image_z_image_turbo.json` | Bundled example workflow copied into the ComfyUI workflow area during installation. It describes a workflow; it does not include the model weights. |
| `docs/ARCH-LINUX.md` | Reproduces the tested Arch/CachyOS-style installation using the official `comfy-cli` workflow. |
| `docs/UBUNTU-SERVER.md` | Adapts the same approach to Ubuntu Server while keeping ROCm and Python compatibility explicit. |
| `docs/GPU-ISOLATION.md` | Explains `HIP_VISIBLE_DEVICES`, dual-GPU assignment, verification, launch templates, and the idle-CPU workaround. |
| `docs/Z-IMAGE-TURBO.md` | Documents the official Z-Image-Turbo files, model folders, workflow, CLI execution, and expected first-run behavior. |
| `docs/TROUBLESHOOTING.md` | Provides a diagnostic order and fixes for GPU selection, ROCm, permissions, idle CPU, ports, model errors, VRAM, and updates. |
| `docs/PORTABILITY.md` | Explains what should transfer to other `gfx1030` RDNA2 cards and which hardware differences can invalidate assumptions. |
| `docs/VERIFIED-LINKS.md` | Lists the upstream repositories and official documentation used by this guide. |
| `docs/REPRODUCTION-CHECKLIST.md` | Gives a concise checklist for reproducing, validating, and safely updating the setup. |

## Scope and privacy

This repository intentionally omits:

- Hostnames, usernames, IP addresses, credentials, and private connection details.
- The network-share mount and the method used to redirect generated output to private storage.
- Any private workflow prompts or generated images.

By default, examples use ComfyUI's ordinary local `output/` directory. Choose your own storage policy separately.

## A simple path through the setup

You do not need to understand every ROCm detail before starting. The easiest way to use this project is to read this section once, then let the helper script take care of the repetitive setup while you make the few decisions that depend on your machine.

### Before you begin

Have these things ready:

- An AMD RDNA2 GPU that reports `gfx1030` through `rocminfo`.
- A supported Linux installation with working ROCm access.
- Enough disk space for ComfyUI, model files, and generated output.
- A decision about where the application should live. The recommended choice is a directory owned by the user who will run ComfyUI, such as `$HOME/ComfyUI`, rather than a system-wide installation.
- The model files you intend to use, downloaded separately from their official sources.

For the documented Z-Image-Turbo example, the useful files are:

- `setup-comfy.sh`, the optional installer in this repository.
- An example workflow JSON such as `image_z_image_turbo.json`, copied into the ComfyUI workflow area by the installer.
- `z_image_turbo_bf16.safetensors`, placed under `models/diffusion_models/`.
- `qwen_3_4b.safetensors`, placed under `models/text_encoders/`.
- `ae.safetensors`, placed under `models/vae/`.

The workflow and model files come from the official upstream sources listed in [Verified upstream links](docs/VERIFIED-LINKS.md). Other models will use different filenames and folders; follow the model's own documentation rather than assuming the Z-Image-Turbo layout applies to everything.

The helper script does not install ROCm, does not download model weights, and does not copy private output destinations. That is intentional: these choices depend on the host, the model license, and how the operator wants to manage storage.

### Experimental installer

`setup-comfy.sh` is an experimental convenience layer around the official `comfy-cli`. It is intended to make the tested AMD setup comfortable for a human to operate, but it is not the authoritative ComfyUI installer and it should not be treated as a replacement for the official CLI documentation.

If you want a standard, transparent installation, follow the official [ComfyUI manual installation guide](https://docs.comfy.org/installation/manual_install) and the official [`comfy-cli` getting-started guide](https://docs.comfy.org/comfy-cli/getting-started). The Arch and Ubuntu documents in this repository explain how those official instructions map to ROCm-backed AMD systems.

If you want to try the convenience installer, download the repository archive and run it as your normal user. This keeps the installer beside the bundled example workflow:

```bash
curl -fsSL https://github.com/GeekyBit/comfy-compatible-amd-linux/archive/refs/heads/main.tar.gz | tar -xz --strip-components=1 && chmod +x setup-comfy.sh && ./setup-comfy.sh
```

You can inspect the repository before running it instead:

```bash
git clone https://github.com/GeekyBit/comfy-compatible-amd-linux.git
cd comfy-compatible-amd-linux
chmod +x setup-comfy.sh
./setup-comfy.sh
```

The installer walks through the setup in plain questions. It asks where the user-run ComfyUI application should be stored, which GPU ComfyUI should see, where models and generated output should live, whether the web page is local to the system or reachable from the local network, and whether the tested idle-CPU workaround should be enabled. It creates a private Python environment for `comfy-cli`, installs the selected ComfyUI version, checks for ROCm-enabled PyTorch, and copies the bundled example workflows without downloading their model files.

The installer checks for Bash, Python with venv support, Git, curl, ROCm, `rocminfo`, `/dev/kfd`, and a render device. If something is missing, it stops and explains what needs to be installed. It does not silently install system packages or use `sudo` for application setup.

After installation, the user-facing interface is one command with simple actions:

```bash
comfy -start
comfy -stop
comfy -status
comfy -setup
comfy -update
comfy -help
```

`comfy -setup` can be run again whenever the storage locations, GPU, web-access scope, port, firewall choice, or optional startup service needs to change. It configures model directories, but it does not choose a specific model. The workflow JSON determines which model a workflow uses.

Local-system mode binds ComfyUI to `127.0.0.1`. Local-network mode lists the host's available LAN addresses, lets the user choose one, reports the browser URL, and checks for an active UFW or firewalld installation. When a firewall is present, the setup asks before adding a rule for the configured port. Existing firewall rules are not silently removed when switching back to local-only mode.

`comfy -update` updates ComfyUI itself while preserving ROCm, PyTorch, model files, and custom nodes. It backs up the launcher configuration, stops ComfyUI if necessary, runs the official `comfy-cli` ComfyUI update, and starts the service again if it was running before the update.

The installer places the `comfy` command at `$HOME/.local/bin/comfy`. If that directory is not already in `PATH`, the installer prints the command needed to add it or you can open a new shell.

The application remains user-run. You do not need to run ComfyUI with `sudo`. If you want it to start automatically when you log in, `comfy -setup` can create a systemd user service and leaves it disabled unless you explicitly enable it.

### After setup

Open the displayed ComfyUI URL in a browser, load a known-good workflow, and run one small test before adding custom nodes or large models. If the result is not what you expect, start with [GPU isolation](docs/GPU-ISOLATION.md) and [troubleshooting](docs/TROUBLESHOOTING.md) rather than changing several packages at once.


## Reference system

The tested reference machine had:

| Component | Reference value |
|---|---|
| GPU | 2 × AMD Radeon Pro V620 |
| GPU architecture | RDNA2, `gfx1030` |
| VRAM | 32 GiB per GPU |
| CPU | Intel Xeon W-2150B, 10 cores / 20 threads |
| RAM | 128 GiB |
| OS | CachyOS, Arch-based Linux |
| ROCm | 7.2.4 |
| Python | 3.12.15 |
| ComfyUI | v0.39.2 |
| `comfy-cli` | 1.22.0 |
| PyTorch | 2.14.1+rocm7.2 |
| PyTorch HIP runtime | 7.2.53211 |

The reference allocation kept ComfyUI on the second V620 while the first V620 was reserved for the LLM service. On a single-GPU system, use logical device `0` instead.

## Tested result

The reference setup successfully:

- Detected the V620 through ROCm/PyTorch.
- Ran ComfyUI through the official `comfy-cli` background manager.
- Generated 1024×1024 Z-Image-Turbo images using an 8-step workflow.
- Produced warm-cache generations in approximately 23–30 seconds on the second V620. First-load time was much longer because model weights had to be loaded.
- Kept ComfyUI isolated from the other GPU with `HIP_VISIBLE_DEVICES`.
- Avoided a ROCr/PyTorch idle busy-spin with `HSA_TOOLS_DISABLE_REGISTER=1` on the reference ROCm stack.

The timing is a measurement from one machine, not a performance guarantee for every `gfx1030` card.

## Quick architecture

```text
ROCm kernel/runtime
        |
PyTorch with HIP support
        |
ComfyUI Python environment
        |
comfy-cli background manager
        |
Z-Image-Turbo workflow
```

For a dual-GPU host:

```text
GPU 0: another workload, such as an LLM
GPU 1: ComfyUI image/video generation
```

Set the ComfyUI device using `HIP_VISIBLE_DEVICES`. Device numbering is remapped after visibility filtering; always verify the selected card with PyTorch and ComfyUI system statistics.

## Installation paths

- [Arch / CachyOS](docs/ARCH-LINUX.md)
- [Ubuntu Server](docs/UBUNTU-SERVER.md)
- [GPU isolation and launch](docs/GPU-ISOLATION.md)
- [Z-Image-Turbo workflow](docs/Z-IMAGE-TURBO.md)
- [Troubleshooting](docs/TROUBLESHOOTING.md)
- [Portability and version policy](docs/PORTABILITY.md)
- [Verified upstream links](docs/VERIFIED-LINKS.md)

## Minimal operating commands

After the installer finishes, use the single wrapper command it installs:

```bash
comfy -start
comfy -stop
comfy -status
comfy -setup
comfy -update
comfy -help
```

`comfy -setup` is where the model root, output directory, GPU visibility, local-system or local-network web access, firewall handling, idle-CPU workaround, and optional startup service are configured. The application directory is chosen during the initial installer run. The wrapper calls the official `comfy-cli` underneath; users do not need to manage the underlying CLI commands for normal operation.

For manual work without the wrapper, use the official CLI executable from its virtual environment explicitly:

```bash
export COMFY_CLI="$HOME/.venvs/comfy-cli/bin/comfy"
"$COMFY_CLI" --workspace="$HOME/ComfyUI" env
```

## Security

Bind to `127.0.0.1` unless remote access is explicitly needed. If remote LAN access is required, bind to the host's LAN address and allow only the trusted LAN or specific client IP in the firewall. ComfyUI authentication was not configured in the reference installation; do not expose an unauthenticated instance directly to the public Internet.

## License

Original installer and documentation files in this repository are copyright (c) GeekyBit and are provided under the [Mozilla Public License 2.0](LICENSE).

This license allows commercial use and private modification while requiring distributed modifications to MPL-covered files to remain available under the MPL. It does not change the licenses of ComfyUI, `comfy-cli`, ROCm, PyTorch, model files, or workflow material obtained from other projects. Check the relevant upstream license before redistributing third-party material.

## Version policy

The reference installation was pinned and tested as a coherent stack. Treat these as a compatibility set, not as independent upgrade targets. Before updating ComfyUI, PyTorch, ROCm, or custom nodes:

1. Save a ComfyUI/custom-node snapshot.
2. Record the current versions.
3. Change one layer at a time.
4. Run the hardware probe and one known-good workflow.
5. Keep the previous environment available for rollback.

Do not casually replace a working ROCm/PyTorch environment with a newer major ROCm release just because it is available upstream.
