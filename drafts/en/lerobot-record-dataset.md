---
title: "LeRobot — Set Up Cameras and Record an Episode Dataset"
emoji: "📝"
type: "tech"
topics: ["lerobot", "robotics", "dataset", "huggingface", "ai"]
published: false
---

> **Series:** LeRobot tutorial — part 3 of 5
> Previous: [Calibration and Teleop (Part 2)](./lerobot-calibration-and-teleoperate) · Next: [Train a Policy (Part 4)](./lerobot-train-policy)

## Introduction

This article walks through everything between "I can teleop the arm" and "I have a dataset I can train on": attaching cameras, verifying their output, configuring them in LeRobot, and recording episodes with `lerobot-record`. The dataset we produce here is the input for [Part 4 (Train a Policy)](./lerobot-train-policy).

We assume the calibration step from [Part 2](./lerobot-calibration-and-teleoperate) is done — leader on `/dev/ttyACM0`, follower on `/dev/ttyACM1`, both with calibration files at `~/.cache/huggingface/lerobot/calibration/`.

## Goal

By the end of this article, you will:

- Identify each connected USB camera and know which physical position it covers (front / wrist / etc.).
- Verify camera resolution, FPS, and pixel format before recording.
- Pass a working `--robot.cameras` configuration to `lerobot-record`.
- Record N episodes with leader teleop and store them under `~/.cache/huggingface/lerobot/<repo_id>/`.
- (Optional) Push the dataset to the Hugging Face Hub.

## Prerequisites

- Ubuntu 22.04 (this article only covers Linux).
- Both SO-101 arms calibrated and working under teleop ([Part 2](./lerobot-calibration-and-teleoperate)).
- At least one USB camera. A second one mounted on the follower wrist is highly recommended.
- A Hugging Face account if you plan to push the dataset (`huggingface-cli login` done once).

## Steps

### 1. Plug in the cameras and check `/dev/video*`

Cameras attach as `/dev/video*` on Linux. Each physical camera typically creates **two** device nodes (one for video, one for metadata) — only one of them is the actual stream.

```bash:terminal
ls /dev/video*
```

To map a node to a physical camera, unplug one and re-run the listing — the nodes that disappear belong to that camera. Note the lowest-numbered node per camera; that is usually the stream you want.

:::message alert
Camera indices can move when you reboot, reconnect, or change which USB port the camera is on. If your "front camera" suddenly looks like the wrist view, that is almost always what happened. For a stable setup, plug cameras into the same ports every time, or pin them by serial number with a `udev` rule.
:::

### 2. Discover cameras with `lerobot-find-cameras`

LeRobot ships a helper that probes all connected cameras and saves one preview JPEG per device, which is the easiest way to confirm "which `/dev/videoN` is which physical view."

```bash:terminal
lerobot-find-cameras opencv
```

Example output (yours will differ):

```text
--- Detected Cameras ---
Camera #0:
  Name: OpenCV Camera @ /dev/video0
  Type: OpenCV
  Id: /dev/video0
  Backend api: V4L2
  Default stream profile:
    Format: 0.0
    Fourcc: YUYV
    Width: 640
    Height: 480
    Fps: 30.0
--------------------
Camera #1:
  Name: OpenCV Camera @ /dev/video2
  Type: OpenCV
  Id: /dev/video2
  Backend api: V4L2
  Default stream profile:
    Format: 0.0
    Fourcc: YUYV
    Width: 640
    Height: 480
    Fps: 30.0
--------------------

Finalizing image saving...
Image capture finished. Images saved to outputs/captured_images
```

Open `outputs/captured_images/` and look at the JPEG from each camera. Decide which one will play the **front** role (overview of the scene) and which will be the **wrist** camera (mounted on the follower).

For this article we will use:

- **front** → `/dev/video0`
- **wrist** → `/dev/video2`

### 3. Pick the resolution and FPS

The defaults (640x480 @ 30 fps, YUYV) are a good starting point and match what most ACT / Diffusion Policy recipes assume. A few things to know before you change them:

- Higher resolution does **not** improve learning much for SO-101 manipulation tasks and quickly saturates USB 2.0 bandwidth (one 1080p MJPEG stream can already starve a second camera on the same bus).
- The `--dataset.fps` flag must match what the cameras can actually deliver. If you ask for 30 fps but the camera only does 15, frames will be dropped or the loop will stall.
- Stick to YUYV or MJPEG that your camera natively supports — software conversion will drop frames.

If you want to confirm what the camera can do at the OS level, `v4l2-ctl` is the tool:

```bash:terminal
v4l2-ctl --device=/dev/video0 --list-formats-ext
```

### 4. Write the record wrapper script

The `lerobot-record` flag list gets long fast — cameras, dataset metadata, and both arms each need their own options. Wrap it once and keep it in version control.

```bash:scripts/record.sh
#!/bin/bash
set -e
source "$(dirname "$0")/../lerobot/.venv/bin/activate"

HF_USER=${HF_USER:-$(whoami)}
DATASET_NAME=${DATASET_NAME:-so101_pick_place}
NUM_EPISODES=${NUM_EPISODES:-50}

lerobot-record \
    --robot.type=so101_follower \
    --robot.port=/dev/ttyACM1 \
    --robot.id=my_follower_arm \
    --robot.cameras="{ front: {type: opencv, index_or_path: /dev/video0, width: 640, height: 480, fps: 30}, wrist: {type: opencv, index_or_path: /dev/video2, width: 640, height: 480, fps: 30} }" \
    --teleop.type=so101_leader \
    --teleop.port=/dev/ttyACM0 \
    --teleop.id=my_leader_arm \
    --display_data=true \
    --dataset.repo_id="${HF_USER}/${DATASET_NAME}" \
    --dataset.single_task="Pick up the cube and place it in the box" \
    --dataset.num_episodes=${NUM_EPISODES} \
    --dataset.episode_time_s=20 \
    --dataset.reset_time_s=10 \
    --dataset.fps=30 \
    --dataset.push_to_hub=false
```

What each block does:

- **`--robot.*`** — the follower (the one that will physically run the task). Cameras attach here because the observation comes from the follower's perspective.
- **`--robot.cameras=...`** — a dict of `{name: {type, index_or_path, width, height, fps}}`. Names (`front`, `wrist`) become column names inside the dataset, so pick something descriptive and keep them stable across recordings of the same task.
- **`--teleop.*`** — the leader. Only used during recording (and teleop testing); at inference time you drop these flags entirely.
- **`--display_data=true`** — opens a Rerun viewer that shows live camera streams and motor state during recording. Extremely useful for spotting "wrist camera is pointing at my hand instead of the workspace" before you waste 50 episodes.
- **`--dataset.repo_id`** — the Hugging Face-style `user/name` slug. Local data is stored under `~/.cache/huggingface/lerobot/<repo_id>/`.
- **`--dataset.single_task`** — the natural-language description of the task. ACT and most language-conditioned policies read this from the dataset.
- **`--dataset.episode_time_s`** / **`reset_time_s`** — see the primer's glossary; these are the recording window and the inter-episode reset window respectively.
- **`--dataset.push_to_hub=false`** — keep it false until you have visually reviewed the dataset.

### 5. Record episodes

Make the script executable and start recording:

```bash:terminal
chmod +x scripts/record.sh
./scripts/record.sh
```

The CLI prints `Recording episode 0`, then opens the Rerun window. Move the leader arm to perform the task. Keyboard controls during a recording session:

| Key | Action |
|-----|--------|
| `→` (right arrow) | End the current episode early and save it. The reset timer starts immediately. |
| `←` (left arrow) | Discard the current episode and re-record it. |
| `Esc` | Stop the entire session. Already-saved episodes are kept. |

Between episodes the CLI shows `Reset the environment` for `--dataset.reset_time_s` seconds — put the cube back, move the box, then the next episode starts automatically.

:::message
You don't have to use the full `episode_time_s`. If you finish the task in 8 seconds, press → and move on. Padding episodes with idle time at the end hurts policy learning; the model spends capacity learning to "do nothing" at the end of every demo.
:::

### 6. Verify the dataset before training

The dataset is written incrementally to:

```
~/.cache/huggingface/lerobot/<repo_id>/
```

Once recording finishes (or you `Esc` out partway), play it back with the visualizer to sanity-check it. A bad dataset costs hours of training; spotting it now costs minutes.

```bash:terminal
python -m lerobot.scripts.visualize_dataset \
    --repo-id "${HF_USER}/so101_pick_place" \
    --episode-index 0
```

Things to check across a handful of episodes:

- Both cameras show the workspace clearly — no blocked views, no auto-exposure flicker.
- The follower actually completes the task (you'd be surprised).
- Episode lengths are consistent. Wildly different lengths mean your reset wasn't consistent.
- No frame drops in the camera streams.

### 7. (Optional) Push to the Hub

Once the dataset looks good, push it:

```bash:terminal
huggingface-cli login    # first time only
```

Then either re-run `record.sh` with `--dataset.push_to_hub=true` for the next batch, or upload the existing folder manually with `huggingface-cli upload`. The pushed dataset can be loaded from any machine with `--dataset.repo_id="${HF_USER}/${DATASET_NAME}"` — the train step in Part 4 will pick it up automatically.

## Troubleshooting

**Error:** `Could not open camera /dev/video0`
**Cause:** Another process is holding the device, or the wrong node was picked (the `+1` metadata node instead of the stream node).
**Fix:** Close other camera consumers (`fuser /dev/video0`), and try the next `/dev/video*` index.

**Symptom:** Recording starts but the Rerun window stays black for one of the cameras.
**Cause:** USB bandwidth is saturated, or the FPS you requested is higher than the camera supports at that resolution.
**Fix:** Move cameras onto different USB controllers (not just different ports on the same hub), or drop the resolution / FPS.

**Symptom:** Episode files are written but `visualize_dataset` fails with a shape mismatch.
**Cause:** You changed `--robot.cameras` mid-recording (e.g. dropped a camera). The dataset schema is fixed when the first episode is written.
**Fix:** Start a new `--dataset.repo_id` or delete the dataset folder under `~/.cache/huggingface/lerobot/` and re-record cleanly.

**Symptom:** Leader feels laggy and follower jerks at low FPS.
**Cause:** Display window and multiple camera decode are competing with the control loop.
**Fix:** Set `--display_data=false` after the first few episodes — once you trust the setup, recording without the viewer is noticeably smoother.

## Summary

- Cameras live at `/dev/video*` on Linux; `lerobot-find-cameras opencv` is the fastest way to map nodes to physical views.
- `--robot.cameras='{ ... }'` takes a dict of named camera configs; the names become the dataset's image columns.
- Recording is leader-teleoperated; `→` saves, `←` redoes, `Esc` stops.
- Datasets land under `~/.cache/huggingface/lerobot/<repo_id>/` and should be played back with `visualize_dataset` before any training run.

## Next

Continue with [Train a Policy (Part 4)](./lerobot-train-policy) — feed this dataset into `lerobot-train` and produce a checkpoint you can run on the robot.

## References

- [LeRobot repository](https://github.com/huggingface/lerobot)
- [LeRobot scripts directory](https://github.com/huggingface/lerobot/tree/main/src/lerobot/scripts) — `lerobot_record.py`, `find_cameras.py`, `visualize_dataset.py`
- [LeRobotDataset v3 format](https://huggingface.co/docs/lerobot/lerobot-dataset-v3)
- [Rerun viewer](https://www.rerun.io/) — used by `--display_data=true`
