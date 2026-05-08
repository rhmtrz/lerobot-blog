---
title: "LeRobot — Install and Calibrate Your Robot Arms"
emoji: "🤖"
type: "tech"
topics: ["lerobot", "robotics", "huggingface", "python", "setup"]
published: false
---

> **Series:** LeRobot tutorial — part 1 of 5
> Previous: [LeRobot Primer (Part 0)](./lerobot-primer-and-tips) · Next: [Teleop control (Part 2)](./lerobot-teleop-control)

## Introduction

This article walks through installing LeRobot in a clean Python environment and calibrating two SO-101 arms (one **leader**, one **follower**) so they can be used for teleoperation, recording, and training in the rest of the series.

We'll cover **two ways to calibrate**: explicit (run `lerobot-calibrate` per arm) and on-demand (let the teleop command prompt you when calibration is missing or stale).

For environment versions, the glossary, and where calibration files are saved, see [the primer](./lerobot-primer-and-tips).

## Goal

By the end of this article, you will:

- Have LeRobot installed in a dedicated virtual environment.
- Know which USB port maps to which arm.
- Have a working calibration file for **each** arm, saved under `~/.cache/huggingface/lerobot/calibration/`.
- Understand the difference between explicit calibration and the teleop-prompted recalibration flow.

## Prerequisites

- Two SO-101 arms (or your variant) connected over USB — one acting as leader, one as follower.
- macOS or Ubuntu.
- Python 3.10+ and [`uv`](https://docs.astral.sh/uv/) installed.
- Read [the primer (Part 0)](./lerobot-primer-and-tips) once.

## Environment

| Component | Version |
|-----------|---------|
| `lerobot` | `<commit or tag>` |
| Python    | `3.10+` |
| OS        | macOS 14 / Ubuntu 22.04 |
| Hardware  | SO-101 ×2 (leader + follower) |

## Steps

### 1. Install LeRobot

Create an isolated venv at `lerobot/.venv` and install the package. Keeping LeRobot in its own venv avoids the "wrong Python environment" gotcha called out in the primer.

```bash:terminal
mkdir -p lerobot && cd lerobot
uv venv .venv
source .venv/bin/activate
uv pip install lerobot
```

Verify the CLI is on your PATH:

```bash:terminal
lerobot --help
lerobot-calibrate --help
```

### 2. Plug in the arms and find the USB ports

Connect both arms over USB, then list serial devices:

```bash:terminal
# macOS
ls /dev/tty.usbmodem*

# Linux
ls /dev/ttyACM* /dev/ttyUSB*
```

You should see two ports — one per arm. To figure out which port is which arm, unplug one and re-list; the missing port is that arm.

In this tutorial:

- **Leader arm** → `/dev/ttyACM0`
- **Follower arm** → `/dev/ttyACM1`

Adjust the commands below to match your ports.

### 3. Calibrate each arm — Option A: pre-calibrate explicitly

Calibration maps each motor's raw encoder values to the physical joint range. Do it once per arm (and again if you re-flash motor firmware).

A small wrapper script keeps the long flag list out of your shell history:

```bash:scripts/calibrate.sh
#!/bin/bash
source "$(dirname "$0")/../lerobot/.venv/bin/activate"

if [ "$1" = "--follower" ] || [ "$1" = "-c" ]; then
    lerobot-calibrate \
        --robot.type=so101_follower \
        --robot.port=/dev/ttyACM1 \
        --robot.id=raha_follower_arm
else
    lerobot-calibrate \
        --robot.type=so101_follower \
        --robot.port=/dev/ttyACM0 \
        --robot.id=raha_leader_arm
fi
```

Make it executable and run once per arm:

```bash:terminal
chmod +x scripts/calibrate.sh
./scripts/calibrate.sh                # leader  (/dev/ttyACM0)
./scripts/calibrate.sh --follower     # follower (/dev/ttyACM1)
```

Follow the on-screen prompts (move each joint through its range, press Enter at the limits). The calibration file is saved to:

```
~/.cache/huggingface/lerobot/calibration/<robot.id>.json
```

> **Tip:** Use a unique `--robot.id` per physical arm (`raha_leader_arm`, `raha_follower_arm` here). The id is the lookup key — every other LeRobot command needs to use the **same** id to find this calibration file.

### 4. Calibrate — Option B: let teleop prompt you

You don't strictly need Option A before running teleop. If teleop starts and detects a missing or stale calibration, it offers to run calibration inline:

```
./scripts/teleoperate.sh
INFO 2026-05-08 13:27:37 eoperate.py:211 {'display_compressed_images': False,
 'display_data': False,
 'display_ip': None,
 'display_port': None,
 'fps': 60,
 'robot': {'calibration_dir': None,
           'cameras': {},
           'disable_torque_on_disconnect': True,
           'id': 'raha_follower_arm',
           'max_relative_target': None,
           'port': '/dev/ttyACM1',
           'use_degrees': True},
 'teleop': {'calibration_dir': None,
            'id': 'raha_leader_arm',
            'port': '/dev/ttyACM0',
            'use_degrees': True},
 'teleop_time_s': None}
INFO 2026-05-08 13:27:37 so_leader.py:72 Mismatch between calibration values in the motor and the calibration file or no calibration file found
Press ENTER to use provided calibration file associated with the id raha_leader_arm, or type 'c' and press ENTER to run calibration:
```

Press `c` then Enter to calibrate right there. This is convenient when you're iterating, but harder to script — for a clean first-time setup, prefer Option A.

### 5. Verify

List the calibration files that were created:

```bash:terminal
ls ~/.cache/huggingface/lerobot/calibration/
```

You should see one JSON file per `--robot.id` you calibrated. If both files exist, you're done — jump to the [Teleop article (Part 2)](./lerobot-teleop-control).

## Troubleshooting

**Error:** `Could not open port /dev/ttyACM0`
**Cause:** OS denied access, or another process is holding the port.
**Fix (Linux):** add your user to `dialout`, then re-login: `sudo usermod -aG dialout $USER`.
**Fix (macOS):** unplug and replug; check System Settings → Privacy & Security for any blocked drivers.

**Error:** `Connected to N motors` where N is less than 6.
**Cause:** Daisy-chain or USB cable problem; one motor unpowered.
**Fix:** Reseat motor cables, check the arm's power LED.

**Calibration prompt loops every run.** If teleop keeps asking to calibrate even after you ran calibration, your `--robot.id` in teleop probably doesn't match the id you calibrated with. They must match exactly — check both commands.

## Summary

- LeRobot installed in `lerobot/.venv` with `uv pip install lerobot`.
- Two arms identified by USB port (`/dev/ttyACM0` leader, `/dev/ttyACM1` follower in this guide).
- Calibrated explicitly with a wrapper script (Option A) or on-demand via teleop (Option B).
- Calibration files live at `~/.cache/huggingface/lerobot/calibration/<robot.id>.json`.

## Next

Continue with [Teleop control (Part 2)](./lerobot-teleop-control) — drive the follower with the leader and confirm everything is wired up correctly.

## References

- [LeRobot repository](https://github.com/huggingface/lerobot)
- [SO-100 / SO-101 hardware setup (TheRobotStudio)](https://github.com/TheRobotStudio/SO-ARM100)
- Series: [Primer (Part 0)](./lerobot-primer-and-tips)
