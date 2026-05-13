---
title: "Building the SO-100 Robot Arm — Assembly Guide"
emoji: "🔧"
type: "tech"
topics: ["lerobot", "robotics", "so100", "hardware", "diy"]
published: false
---

> **Related:** [LeRobot Primer (Part 0)](./lerobot-primer-and-tips) · [Installation & Calibration (Part 1)](./lerobot-install-and-calibration)
> This article is the hardware prerequisite for the LeRobot tutorial series — finish it before installing LeRobot.

## Introduction

This article walks through building two SO-100 arms from a kit: one **Leader** (the one a human moves by hand during teleop) and one **Follower** (the one that actually executes the task). At the end, both arms will be assembled, wired, and recognized over USB so the rest of the LeRobot series can use them. Reader assumption: comfortable with a screwdriver and a terminal, but new to robot-arm assembly.

For the Leader/Follower concept and the teleop flow, see [the primer](./lerobot-primer-and-tips). This article focuses purely on the hardware build.

## Goal

By the end of this article, you will have:

- A Leader arm assembled with gears removed (low-friction for hand operation).
- A Follower arm assembled with powered gripper.
- Six STS3215 motors per arm flashed with unique IDs (`1`–`6`) and connected in a daisy chain.
- Both arms enumerating on USB and responding to `lerobot-setup-motors`.

Total kit cost is around **$123 USD / arm** ($246 for the pair) plus filament and the ~6 hours of print time per arm.

## Prerequisites

### Tools

- Small flat-head screwdriver (for popping support material off 3D prints).
- Precision Phillips screwdriver set.
- Hex/Allen key set — `<TBD: confirm sizes used in SO-100>`.
- 3D printer (or a print service that accepts STL).
- Optional: 2–4 bench clamps. The shoulder and elbow joints are awkward to hold steady during screw-down — clamps make the first arm much less frustrating.
- Optional: ESD mat and a small tray for screws.

### 3D-printed parts

All STLs live in the [SO-ARM100 repository](https://github.com/TheRobotStudio/SO-ARM100). Print settings from the official `SO100.md`:

| Setting | Value |
|---------|-------|
| Material | PLA or PLA+ |
| Nozzle | 0.4 mm (or 0.6 mm) |
| Layer height | 0.2 mm (0.4 mm with 0.6 mm nozzle) |
| Infill | 13–15 % |
| Supports | Yes — orientation files included for Ender 3 and Prusa MINI+ |

Print one full set per arm. Expect roughly `<TBD: total filament / hours from your slicer>` per set.

:::message
Remove **all** support material before assembly. The motor brackets and horn pockets are tight to begin with — leftover support strands turn a 5-minute step into a 30-minute one.
:::

### Bill of Materials (per arm)

| Category | Item | Qty | Notes |
|----------|------|-----|-------|
| Motors | Feetech **STS3215** servo | 6 | Default factory ID is `1` for all motors — you must re-flash each one. |
| Control board | Waveshare USB-C servo driver | 1 | Set the on-board jumper to **B** (USB) before powering on. |
| Cables | 3-pin servo cables | 6 | Daisy-chained from motor 1 → motor 6. Lengths: `<TBD: check kit>`. |
| Power supply | DC barrel jack PSU | 1 | `<TBD: confirm voltage/amperage from your kit — check the Waveshare board's input spec>`. |
| Fasteners | Mixed screws | per BOM | Counts are per assembly step in the official docs; sizes: `<TBD>`. |
| Motor horns | Included with STS3215 | — | One horn per motor; the gripper uses a single horn. |

The full canonical BOM and STL set is in [`SO-ARM100/SO100.md`](https://github.com/TheRobotStudio/SO-ARM100/blob/main/SO100.md). When the official docs and this article disagree, trust the upstream repo.

### Workspace

- Flat, well-lit surface.
- Container or magnetic tray for small screws (you will drop them).
- USB-C cable and a host machine running macOS or Ubuntu for the motor-ID step.

## Step-by-step assembly

### 1. Print and de-support the parts

Slice the STLs with the settings above and print one set. Once cool, pop off support material with a small flat-head screwdriver. Test-fit each motor into its bracket before assembly — if it doesn't slide in cleanly, file or scrape the inner walls. **Do this now**, while the bracket is still standalone, not after it's bolted to the arm.

### 2. Set motor IDs *before* assembling anything mechanical

This is the single most important step. Every STS3215 ships with the default ID `1`. The arm's daisy chain needs IDs `1`–`6` (base → gripper). And — critically — motor connectors are buried inside the brackets after final assembly; if you skip this and assemble first, you'll be partially disassembling the arm to fix it.

Connect the Waveshare board to your host with USB-C, set its jumper to **B**, then plug exactly **one** motor in at a time.

Find the serial port:

```bash:terminal
lerobot-find-port
```

You should see one port that appears when you plug the board in and disappears when you unplug it. On macOS that's typically `/dev/tty.usbmodem...`; on Linux it's `/dev/ttyACM0` or `/dev/ttyUSB0`.

Then flash IDs one motor at a time:

```bash:terminal
lerobot-setup-motors \
    --robot.type=so100_follower \
    --robot.port=/dev/ttyACM0
```

Follow the prompts — the script asks you to plug in motors in order (1 = base/shoulder pan, 6 = gripper). Repeat the entire process for the Leader using `--robot.type=so100_leader`.

:::message
**Linux only:** if you get `Permission denied` opening the serial port, either add yourself to the `dialout` group (`sudo usermod -aG dialout $USER`, then log out and back in) or run `sudo chmod 666 /dev/ttyACM0` for the current session.
:::

:::message alert
Don't skip this and "figure it out after assembly." Re-flashing a motor that's already inside the arm requires removing the bracket holding it. Set IDs now, while every motor is still free in your hand.
:::

### 3. Build the base and shoulder pan (motor 1)

- Set motor 1 into the base motor bracket.
- Secure the motor to the bracket with `<TBD: screw count>` screws.
- Attach the motor horn to the output shaft.
- Bolt the bracket to the printed base plate.

This is the foundation — the rest of the arm cantilevers off it. Don't skip the base plate screws thinking you'll do them later; the arm gets harder to flip over with each joint you add.

### 4. Add the shoulder roll (motor 2) and upper arm

- **Pre-route the cable** from motor 1 through the channel in the motor 2 bracket before fastening anything.
- Mount motor 2 in its bracket, attach the horn, then secure to the shoulder pan output.
- Attach the upper-arm structural prints to motor 2's horn with `<TBD: screws per side>`.

The "pre-route the cable" advice repeats for every joint from here on. It's faster to thread a cable through an empty bracket than to fish one through after the bracket is sandwiched in place.

### 5. Add the elbow (motor 3) and forearm

- Pre-route the cable from motor 2.
- Mount motor 3, attach the horn, bolt to the upper-arm output.
- Attach the forearm prints.

### 6. Add the wrist (motors 4 and 5)

- Motor 4 is the wrist pitch — mount it on the forearm output.
- Motor 5 is the wrist roll — mount it on motor 4's output. The wrist roll uses a single motor horn.
- Pre-route both cables.

### 7. End-effector — Follower or Leader

This step diverges depending on which arm you're building. Build the Follower first; it's the simpler of the two.

#### 7a. Follower: gripper

- Mount motor 6 in the gripper holder.
- Attach the gripper claw to motor 6's horn with `<TBD: screw count>` screws.
- Connect the 3-pin cable from motor 5 → motor 6.

Motor 6 on the Follower is a normal powered servo — it opens and closes the claw when the policy commands it.

#### 7b. Leader: gear removal and trigger handle

The Leader is **passive** — a human moves it by hand. To make that feel natural, you remove the internal gears from each STS3215 so the motors spin freely and only report position. Refer to the gear-removal video linked from the [Hugging Face SO-100 docs](https://huggingface.co/docs/lerobot/en/so100); doing it well takes practice.

:::message alert
Gear removal is largely irreversible without spare parts, and the motor will no longer behave as a powered servo afterwards. Don't remove gears from motors you intend to use on the Follower. Do all six Leader motors **before** you start assembling the Leader's mechanical structure.
:::

After gear removal:

- Mount the wrist-mounted Leader holder on motor 5 (no gripper claw).
- Attach the trigger handle to the holder with `<TBD: screw count>` screws.
- Mount motor 6 in the trigger assembly and attach the trigger lever to its horn.

Motor 6 on the Leader is also gearless — it reports the trigger position so the Follower's gripper can mirror it.

### 8. Mount the controller and daisy-chain the motors

- Mount the Waveshare board to the back of the base with `<TBD: screw count>` screws.
- Connect the 3-pin daisy-chain in order: board → motor 1 → motor 2 → … → motor 6.
- Plug in the power supply to the Waveshare board.

:::message alert
Do not power on with any motor cable disconnected or with the jumper on the wrong setting. A loose connector at power-up can put a motor in an undefined state and require reflashing.
:::

### 9. Power on and smoke-test

Plug in the USB cable and power, then check that the OS sees the controller:

```bash:terminal
# macOS
ls /dev/tty.usbmodem*

# Linux
ls /dev/ttyACM* /dev/ttyUSB*
```

You should see one new port per controller you've plugged in. Then confirm all six motors respond:

```bash:terminal
lerobot-setup-motors \
    --robot.type=so100_follower \
    --robot.port=/dev/ttyACM0
```

If the script reports `Connected to 6 motors`, the arm is ready for calibration. Repeat for the Leader.

## Troubleshooting

**Error:** `Connected to N motors` where N < 6
**Cause:** Broken or unseated 3-pin cable somewhere in the chain, or a missing motor ID.
**Fix:** Power off, then reseat every cable from the controller out. If reseating doesn't help, suspect an ID collision — disconnect motors one at a time to find the missing/duplicate ID. Worst case, partial disassembly to re-flash one motor.

**Motor not detected at all after assembly**
**Cause:** ID was never set, or the motor was flashed as Leader but installed in the Follower (or vice versa).
**Fix:** Disconnect that motor's bracket, flash with `lerobot-setup-motors` against the correct `--robot.type`, reinstall.

**Leader joints feel stiff / require force**
**Cause:** Gears not fully removed from one or more motors.
**Fix:** Identify the stiff joint by feel, remove that motor, finish the gear removal, reinstall.

**Error:** `Could not open port /dev/ttyACM0` (Linux)
**Cause:** Your user isn't in the `dialout` group, or another process is holding the port.
**Fix:** `sudo usermod -aG dialout $USER` then re-login, or `sudo chmod 666 /dev/ttyACM0` for the current session.

**The first arm took 3 hours and I'm exhausted**
**Cause:** Normal. The official docs note the first arm takes >1 hour; the second drops well below an hour.
**Fix:** Take a break before starting the second arm. Don't try to finish both in one sitting — fatigue mistakes during gear removal are the worst kind.

## Summary

- Two arms (Leader + Follower) assembled from one BOM each, ~$123/arm.
- Six STS3215 motors per arm, flashed with unique IDs `1`–`6` **before** mechanical assembly.
- Leader's motors have gears removed so a human can move it freely; Follower's are intact.
- Waveshare controller wired in a daisy chain, board jumper set to **B**, USB enumerating on the host.
- Smoke-tested with `lerobot-setup-motors` reporting `Connected to 6 motors` on both arms.

## Next

Install LeRobot and calibrate both arms in [Installation & Calibration (Part 1)](./lerobot-install-and-calibration).

## References

- [SO-ARM100 hardware repository](https://github.com/TheRobotStudio/SO-ARM100) — STL files, BOM, official assembly notes.
- [`SO100.md`](https://github.com/TheRobotStudio/SO-ARM100/blob/main/SO100.md) — print settings, motor model, cost.
- [Hugging Face SO-100 docs](https://huggingface.co/docs/lerobot/en/so100) — step-by-step assembly with photos, motor setup commands, Leader/Follower differences.
- [LeRobot repository](https://github.com/huggingface/lerobot) — `lerobot-find-port`, `lerobot-setup-motors`, `lerobot-calibrate`.
- [LeRobot Primer (Part 0)](./lerobot-primer-and-tips) — Leader/Follower roles, teleop concepts, log reference.
