# Z-Image-Turbo workflow

The reference installation used the official Comfy-Org Z-Image-Turbo model files and workflow.

## Model placement

Place the official files in the matching ComfyUI model folders:

```text
ComfyUI/
├── models/
│   ├── diffusion_models/
│   │   └── z_image_turbo_bf16.safetensors
│   ├── text_encoders/
│   │   └── qwen_3_4b.safetensors
│   └── vae/
│       └── ae.safetensors
```

The [official ComfyUI Z-Image-Turbo tutorial](https://docs.comfy.org/tutorials/image/z-image/z-image-turbo) links to the exact files and explains their destination folders; the files are hosted in the [official Comfy-Org model repository](https://huggingface.co/Comfy-Org/z_image_turbo).

Do not rename the files unless the workflow is updated to match the new names.

## Workflow

Use the official workflow template:

- `image_z_image_turbo.json`
- Official source: Comfy-Org's workflow template repository

Load the JSON into the ComfyUI web UI, confirm that the model nodes resolve, enter a prompt, and queue the workflow.

The reference test used:

```text
Resolution: 1024 × 1024
Steps: 8
GPU: second Radeon Pro V620
```

Z-Image-Turbo is a distilled workflow; the step count is part of the intended fast-generation setup. Higher step counts may change quality and speed but are not the reference test.

## CLI execution

The repository's installed `comfy` command is the operator wrapper. For direct workflow execution with the underlying official CLI, use its explicit virtual-environment path:

```bash
export COMFY_CLI="$HOME/.venvs/comfy-cli/bin/comfy"
```

ComfyUI API-format workflows can be run through `comfy-cli` after the server is healthy:

```bash
export COMFY_LOCAL_URL=http://127.0.0.1:8188
"$COMFY_CLI" --workspace="$COMFYUI_HOME" run \
  --workflow ./workflows/image_z_image_turbo.json \
  --wait
```

Use a workflow exported in API format. A normal UI workflow export and an API-format workflow are not always interchangeable.

## First run versus warm cache

The first generation is expected to be much slower because ComfyUI loads the diffusion model, text encoder, and VAE. Measure a second generation after the models are resident before diagnosing a performance regression.

Record:

- Resolution and batch size.
- Steps and sampler/scheduler.
- First-run or warm-cache state.
- GPU selected by `system_stats`.
- Generation time.
- VRAM and CPU usage.

## Output policy

This guide deliberately documents only ComfyUI's normal local output behavior. Storage redirection, network mounts, synchronization, and private share paths are deployment-specific and omitted.

## Common workflow failures

### Missing diffusion model

Confirm the file exists in `models/diffusion_models/` and restart or refresh ComfyUI's model list.

### Missing text encoder or VAE

Confirm the exact filenames and folders. The nodes must reference `qwen_3_4b.safetensors` and `ae.safetensors` in their respective directories.

### Out of memory

Reduce resolution or batch size first. Confirm that the process is on the intended 32 GiB GPU and that no other workload is using it. Do not solve a VRAM problem by exposing both GPUs unless the workflow explicitly supports multi-GPU execution.

### Workflow imports but does not queue

Check the ComfyUI log and verify that all nodes are present. Export the workflow again in API format if using `comfy run`.
