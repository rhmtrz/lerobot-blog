---
title: "<English title — replace before commit>"
emoji: "📚"
type: "tech"
topics: ["lerobot", "robotics", "huggingface", "ai", "python"]
published: false
---

> **Series:** LeRobot tutorial — Part 0 (prerequisite reference)
> Read this once, then refer back to it from every other article in the series.

## Why this article exists

Things I wish I had known on day one. Concepts, environment details, log lines, and paths that come up over and over — explained once here, so the tutorial articles can stay focused on doing.

If a tutorial step says "see the primer," this is what it means.

## Environment cheat sheet

| Item | Value |
|------|-------|
| OS | macOS / Ubuntu (specify versions you tested) |
| Python | 3.10+ |
| Package manager | `uv` / `pip` |
| `lerobot` version | `<commit or tag>` |
| Hardware | `<SO-100 / SO-ARM100 / Koch>` |

How to check each one:

```bash:terminal
python --version
pip show lerobot | head -3
uname -a
```

## Hardware quick reference

### USB and serial ports

How LeRobot finds your robot. The arm shows up as `/dev/tty.usbmodem*` (macOS) or `/dev/ttyACM*` / `/dev/ttyUSB*` (Linux).

```bash:terminal
# macOS
ls /dev/tty.usbmodem*

# Linux
ls /dev/ttyACM* /dev/ttyUSB*
```

### Motor IDs

Each Dynamixel/Feetech motor on a LeRobot arm has an **ID** (1–6 typically). The ID is written into the motor's EEPROM and survives power cycles. If you ever swap a motor, you must re-assign its ID with the LeRobot motor setup script.

### Cameras

Cameras are identified by **index** (`0`, `1`, ...) on macOS, or **device path** (`/dev/video0`) on Linux. The index is **not stable** across reboots or USB hub changes. Pin cameras by their serial number when you can.

```bash:terminal
# List cameras (macOS)
system_profiler SPCameraDataType

# List cameras (Linux)
v4l2-ctl --list-devices
```

## Data paths

Where LeRobot stores things by default:

| What | Path |
|------|------|
| Hugging Face cache (datasets, models) | `~/.cache/huggingface/` |
| LeRobot dataset cache | `~/.cache/huggingface/lerobot/` |
| Calibration files | `~/.cache/huggingface/lerobot/calibration/` |
| Local checkpoints (training output) | `outputs/train/<run-name>/` |

Always check these paths before assuming a record/train command failed silently — usually it just wrote the data somewhere you didn't look.

## Glossary

**Episode** — One complete demonstration of a task, from start to finish. *Not* a continuous recording. Recording 50 episodes means 50 separate start-to-end demos. Between episodes, you reset the environment (move the object back, etc.).

**Reset time** — The pause between episodes during recording. The CLI tells you when to reset; the next episode starts after the timer.

**Episode duration** — Maximum length of one episode in seconds. The recording auto-stops at this limit; you can also stop early.

**Dataset** — A collection of episodes, plus metadata (camera streams, motor positions, actions) saved in a Hugging Face-compatible format and pushable to the Hub.

**Teleop (teleoperation)** — Driving a "follower" robot live with a "leader" device (another arm, joystick, etc.). Used to record demonstrations.

**Policy** — The trained model that maps observations → actions. ACT, Diffusion Policy, TDMPC, Pi0 are policy types in LeRobot.

**Calibration** — The procedure that maps raw motor encoder values to physical joint angles. Required once per arm; saved to disk.

**Checkpoint** — A snapshot of the policy weights during training. Resume training, or load for inference, from a checkpoint.

**Inference** — Running the trained policy live on the robot. Same hardware as recording, but the policy issues actions instead of you.

## Reading the logs

Common log lines and what they actually mean:

| Log line | Meaning |
|----------|---------|
| `Recording episode N` | Starting episode N. Begin demonstrating the task. |
| `Reset the environment` | Episode finished. Move the workspace back to the start state for episode N+1. |
| `Episode duration reached` | The per-episode timer hit `--control.episode_time_s`. Recording stopped automatically. |
| `Calibration not found` | No calibration file at the expected path — run the calibration step first. |
| `Connected to motor X` | A motor handshake succeeded. If you see fewer motor connect lines than motors, your USB or daisy-chain cable has an issue. |
| `Pushing dataset to hub` | Uploading to Hugging Face. Needs `huggingface-cli login` to have run. |

When in doubt, the answer is in the log — read line by line, don't skim.

## Common gotchas

- **USB hub flakiness.** A passive hub can drop motor traffic under load. Use a powered hub or plug the arm directly into the laptop.
- **Camera index drift.** Adding/removing a USB device shifts indices. If your front camera suddenly looks like the wrist camera, this is why.
- **Calibration after firmware change.** Re-flashing motors invalidates calibration. Recalibrate.
- **Mixing Python environments.** `pip install lerobot` in one venv and running `lerobot ...` from another is the #1 cause of "command not found" or "wrong version" issues.
- **Dataset path collisions.** Recording with the same `--repo-id` overwrites or appends — read the prompt the CLI gives you.

## How to learn LeRobot effectively

- **Read every CLI flag once.** `lerobot record --help`, `lerobot train --help`. Most "magic" disappears once you skim the flags.
- **Watch the dataset before you train.** `lerobot.scripts.visualize_dataset` confirms the data is what you think it is.
- **Train the smallest config first.** Cut episode count, batch size, training steps — get the full pipeline working end-to-end before optimizing.
- **Save logs.** Pipe `tee` on every record/train run. When something fails three days later, you'll thank yourself.

## References

- [LeRobot repository](https://github.com/huggingface/lerobot)
- [LeRobot examples](https://github.com/huggingface/lerobot/tree/main/examples)
- [Hugging Face datasets viewer](https://huggingface.co/datasets)
- Series articles 1–5 (linked below as they're published)
