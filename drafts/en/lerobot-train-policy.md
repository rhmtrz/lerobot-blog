---
title: "LeRobot — Train a Policy on Your Recorded Dataset"
emoji: "🧠"
type: "tech"
topics: ["lerobot", "ai", "machinelearning", "pytorch", "robotics"]
published: false
---

> **Series:** LeRobot tutorial — part 4 of 5
> Previous: [Record an Episode Dataset (Part 3)](./lerobot-record-dataset) · Next: [Run a Trained Model (Part 5)](./lerobot-run-trained-model) *(coming soon)*

## Introduction

You have a recorded dataset — leader trajectories, follower joint states, one or two camera streams, all bundled in `LeRobotDataset` format. This article takes that dataset and produces a **policy checkpoint** that can later be run on the same robot.

We will use **ACT (Action-Chunking Transformer)** as the default policy. It is the easiest starter for SO-101 with ~50 episodes and the one most people use first before reaching for Diffusion Policy or Pi0.

## Goal

By the end of this article, you will:

- Understand which policy type to pick for a small SO-101 dataset.
- Run `lerobot-train` and read the training logs.
- Know where checkpoints are written and how to resume from one.
- Optionally push the trained policy to the Hugging Face Hub for use on another machine.

## Prerequisites

- A dataset from [Part 3](./lerobot-record-dataset) — ideally 30–50 episodes of the same task.
- Ubuntu 22.04 (this article only covers Linux).
- An NVIDIA GPU with CUDA installed. ACT trains comfortably on an RTX 3060 (12 GB) or better. macOS with Apple silicon (MPS) works but is slower and occasionally hits operator-coverage issues — covered in [Troubleshooting](#troubleshooting).
- (Optional) A free [Weights & Biases](https://wandb.ai/) account if you want loss curves in a browser instead of just terminal logs.

## Environment

| Component | Version |
|-----------|---------|
| `lerobot` | `<commit or tag>` |
| Python    | `3.10+` |
| CUDA      | `12.x` (matches your PyTorch wheel) |
| GPU       | ≥ 8 GB VRAM recommended for ACT at default settings |
| Hardware  | SO-101 follower (only needed at inference time — not here) |

## Steps

### 1. Pick a policy

For a first run, use **ACT**. Reasons:

- Designed for SO-100/SO-101-style bimanual / single-arm imitation tasks.
- Converges with as few as 30 episodes.
- Pure imitation — no reward signal, no environment, no rollouts during training.
- Default hyperparameters in LeRobot are sensible; you usually don't have to touch them.

Alternatives, in rough "try later" order:

- **Diffusion Policy** — slightly more accurate on long-horizon tasks, but wants more episodes (~150+) and trains slower.
- **Pi0** / **SmolVLA** — pretrained foundation policies. Useful if you're language-conditioning across multiple tasks. Fine-tuning still uses `lerobot-train`, but with a base checkpoint flag.

### 2. (Optional) Log in to Weights & Biases

If you want loss curves and gradient plots in the browser:

```bash:terminal
pip install wandb
wandb login
```

Then add `--wandb.enable=true --wandb.project=<your-project>` to the training command below. Skip this entirely if you're happy reading the terminal output.

### 3. Write the train wrapper script

`lerobot-train` accepts a long list of flags. Wrap them once so the command is reproducible and the parameters are version-controlled:

```bash:scripts/train.sh
#!/bin/bash
set -e
source "$(dirname "$0")/../lerobot/.venv/bin/activate"

HF_USER=${HF_USER:-$(whoami)}
DATASET_NAME=${DATASET_NAME:-so101_pick_place}
RUN_NAME=${RUN_NAME:-act_${DATASET_NAME}_$(date +%Y%m%d_%H%M)}

lerobot-train \
    --dataset.repo_id="${HF_USER}/${DATASET_NAME}" \
    --policy.type=act \
    --policy.device=cuda \
    --output_dir="outputs/train/${RUN_NAME}" \
    --job_name="${RUN_NAME}" \
    --batch_size=8 \
    --steps=100000 \
    --save_freq=10000 \
    --log_freq=200 \
    --wandb.enable=false \
    --policy.push_to_hub=false
```

What each flag does:

- **`--dataset.repo_id`** — same id you used in Part 3. If the dataset is local, LeRobot finds it under `~/.cache/huggingface/lerobot/<repo_id>/`. If it's on the Hub, LeRobot pulls it.
- **`--policy.type=act`** — selects ACT. Other valid values include `diffusion`, `pi0`, `smolvla`, `tdmpc`.
- **`--policy.device=cuda`** — use the NVIDIA GPU. `mps` for Apple silicon, `cpu` as a fallback (slow).
- **`--output_dir`** — everything for this run is written under here: config, checkpoints, logs.
- **`--job_name`** — appears in logs and in the W&B run name. Make it unique per run to avoid mix-ups.
- **`--batch_size=8`** — safe default for ~8 GB VRAM. Bump to 16 or 32 if you have headroom.
- **`--steps=100000`** — number of gradient steps, *not* epochs. 100k is a reasonable default for ACT on ~50 episodes.
- **`--save_freq=10000`** — save a checkpoint every N steps. Lower = more recovery points, more disk usage.
- **`--log_freq=200`** — how often to print loss to the terminal.
- **`--policy.push_to_hub=false`** — only flip to true once you've verified the policy works on the robot (Part 5).

### 4. Run training

```bash:terminal
chmod +x scripts/train.sh
./scripts/train.sh 2>&1 | tee outputs/train/last_run.log
```

Piping through `tee` saves the full log to a file — useful when something goes wrong overnight and you want to scroll back without `screen` / `tmux`.

You should see, in order:

1. The full resolved config (every default plus your overrides). Skim it once to confirm the dataset path, policy type, and device are what you intended.
2. Dataset loading: `Loaded dataset with N episodes`.
3. Step logs at `--log_freq` cadence:

```text
step: 200  loss: 0.812  grad_norm: 1.43  lr: 1.0e-05  update_s: 0.124  data_s: 0.041
step: 400  loss: 0.534  grad_norm: 0.91  lr: 1.0e-05  update_s: 0.118  data_s: 0.038
...
```

Loss should drop fast in the first few thousand steps, then settle. Plateaus around 0.05–0.15 are normal for ACT on small datasets.

4. Checkpoint saves every `--save_freq` steps:

```text
INFO  Saving checkpoint at step 10000 to outputs/train/<RUN_NAME>/checkpoints/010000/
```

### 5. Where checkpoints land

The output directory tree:

```
outputs/train/<RUN_NAME>/
├── config.json
├── checkpoints/
│   ├── 010000/
│   │   ├── pretrained_model/
│   │   │   ├── config.json
│   │   │   └── model.safetensors
│   │   └── training_state/
│   │       ├── optimizer.bin
│   │       └── scheduler.bin
│   ├── 020000/
│   ├── ...
│   └── last/ -> ../checkpoints/100000/
└── logs/
```

`pretrained_model/` is what Part 5 (and `lerobot-record --policy.path=...`) loads at inference time. `training_state/` is only needed if you want to resume training.

### 6. Resume from a checkpoint

If training was interrupted (power, OOM, ssh dropped), resume from the latest checkpoint:

```bash:terminal
lerobot-train \
    --config_path="outputs/train/${RUN_NAME}/checkpoints/last/pretrained_model/" \
    --resume=true
```

`--resume=true` re-loads optimizer and scheduler state from `training_state/`, so the run continues from exactly where it stopped (same LR schedule, same step count, same seed if you set one).

### 7. (Optional) Push to the Hub

Once you're happy with the loss curve and want the policy on another machine:

```bash:terminal
huggingface-cli login   # first time only
```

Then either re-run `train.sh` with:

```
--policy.push_to_hub=true \
--policy.repo_id="${HF_USER}/act_so101_pick_place"
```

…or upload the existing `pretrained_model/` folder by hand:

```bash:terminal
huggingface-cli upload \
    "${HF_USER}/act_so101_pick_place" \
    "outputs/train/${RUN_NAME}/checkpoints/last/pretrained_model/"
```

## Troubleshooting

**Error:** `torch.cuda.OutOfMemoryError: CUDA out of memory.`
**Cause:** Batch size too large for the GPU.
**Fix:** Drop `--batch_size` (try 4, then 2). If that's still too tight, ACT supports gradient accumulation in newer versions; check `lerobot-train --help` for the current flag.

**Symptom:** Loss decreases for a few thousand steps, then flatlines high (> 0.5).
**Cause:** The dataset is the bottleneck, not the optimizer. Either too few episodes, inconsistent demonstrations, or a camera was misframed.
**Fix:** Replay several episodes with `visualize_dataset` (Part 3, step 6) and confirm they look like the task. Re-record the bad ones if needed. Then re-run training from scratch on the cleaned dataset.

**Symptom:** On macOS, training fails with a missing MPS operator (`aten::...` not implemented).
**Cause:** PyTorch's MPS backend doesn't yet implement every op ACT uses.
**Fix:** Set `--policy.device=cpu` to confirm the rest of the pipeline works, then either move training to a Linux+CUDA box or wait for the MPS backend to catch up. Inference on MPS usually works even when training doesn't.

**Symptom:** `KeyError: 'observation.images.front'` (or similar) when loading the dataset.
**Cause:** Camera name in the dataset doesn't match what the policy expects. Usually means the cameras dict in `record.sh` and the policy config disagree.
**Fix:** Re-record with consistent camera names, or pass `--policy.input_features=...` overrides to match what's in the dataset.

**Symptom:** `RuntimeError: shape mismatch ...` on the first training step.
**Cause:** Calibration drifted between record and train (different `--robot.id`, or a re-flashed motor changed the action dim).
**Fix:** Re-check the dataset's `meta/info.json` — `action.shape` and `observation.state.shape` should match what the policy was configured for. If they don't, re-record with the corrected calibration.

:::message
Long error tracebacks are usually pasted in full when asking for help; wrap them in `:::details` blocks in your own notes so they don't dominate the page.
:::

## Summary

- ACT is the default policy for SO-101 with ~50 episodes; reach for Diffusion Policy or Pi0 only after ACT works.
- Training is driven by `lerobot-train` with `--policy.type`, `--dataset.repo_id`, `--output_dir`, `--steps`, and `--batch_size` as the main knobs.
- Checkpoints land in `outputs/train/<run>/checkpoints/<step>/pretrained_model/`, with a `last/` symlink for the most recent one.
- Use `--resume=true` to pick up an interrupted run, and `--policy.push_to_hub=true` only after the policy is validated.

## Next

Part 5 (running the trained model on the robot — i.e., evaluating it) will be a separate article. The short version, for reference, is that `lerobot-record` accepts a `--policy.path=...` flag that takes the same `pretrained_model/` folder we produced here.

## References

- [LeRobot repository](https://github.com/huggingface/lerobot)
- [LeRobot scripts directory](https://github.com/huggingface/lerobot/tree/main/src/lerobot/scripts) — `lerobot_train.py`
- [LeRobot policies overview](https://huggingface.co/docs/lerobot/policies) — ACT, Diffusion Policy, Pi0, SmolVLA
- [Weights & Biases integration](https://docs.wandb.ai/)
