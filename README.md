# LeRobot Piper for LeRobot 0.4.3

This repository is a LeRobot 0.4.3 refactor of the Piper robot integration.

The Piper-specific implementation was written with reference to
[WeGo-Robotics/lerobot_piper](https://github.com/WeGo-Robotics/lerobot_piper.git),
which provided Piper support for LeRobot 0.3.3. This project adapts that work to
the LeRobot 0.4.3 codebase and CLI structure while preserving the normal LeRobot
workflow for teleoperation, dataset recording, policy training, and evaluation.

This is intended for AgileX Piper leader-follower setups using Linux SocketCAN
interfaces.

## What This Repository Adds

- A Piper motor bus implementation at `src/lerobot/motors/piper/`
- A LeRobot `Robot` implementation for the follower arm:
  `--robot.type=piper_follower`
- A LeRobot `Teleoperator` implementation for the leader arm:
  `--teleop.type=piper_leader`
- Piper registration in the LeRobot robot and teleoperator factory utilities
- SocketCAN setup helpers for leader/follower CAN interfaces
- Example scripts for teleoperation, dual-port recording, single-port recording,
  and HIL joint-delta recording

The rest of the repository follows the upstream Hugging Face LeRobot 0.4.3
layout and command-line tools.

## Main Components

```text
src/lerobot/motors/piper/
  piper.py                         PiperMotorsBus wrapper around the Piper SDK
  tables.py                        Piper motor model and initialization tables

src/lerobot/robots/piper_follower/
  config_piper_follower.py         LeRobot robot config registration
  piper_follower.py                Follower arm, cameras, observations, actions

src/lerobot/teleoperators/piper_leader/
  config_piper_leader.py           LeRobot teleoperator config registration
  piper_leader.py                  Leader arm action source

src/lerobot/scripts/
  lerobot_record_singleport.py     Direct/single-port recording flow
  lerobot_record_HIL_joint_delta.py HIL recording flow with a policy in the loop

1_init_can.sh                      CAN interface setup and renaming helper
2_teleop.sh                        Leader-follower teleoperation example
3_doubleport_record.sh             Dual-CAN dataset recording example
4_singleport_record.sh             Single-CAN/direct recording example
5_HIL_record.sh                    HIL joint-delta recording example
```

## Refactor Notes

Compared with the LeRobot 0.3.3-oriented Piper code referenced above, this
repository updates the integration for LeRobot 0.4.3 patterns:

- `PiperFollowerConfig` is registered with `RobotConfig.register_subclass`.
- `PiperLeaderConfig` is registered with `TeleoperatorConfig.register_subclass`.
- `PiperFollower` follows the 0.4.3 `Robot` interface for observations, actions,
  camera configs, and safe action limiting.
- `PiperLeader` follows the 0.4.3 `Teleoperator` interface for action features
  and leader-arm reads.
- `PiperMotorsBus` adapts Piper SDK reads/writes into LeRobot motor bus semantics,
  including normalized joint and gripper positions.
- The factory helpers can instantiate Piper devices from CLI options such as
  `--robot.type=piper_follower` and `--teleop.type=piper_leader`.

## Requirements

- Ubuntu/Linux environment with SocketCAN support
- Python 3.10 or newer
- Two CAN adapters for the default leader-follower examples
- AgileX Piper arm hardware
- Piper SDK Python packages available in the environment, including the modules
  imported by the driver:
  - `piper_sdk`
  - `wego_piper`
- Camera devices if recording vision datasets
- A Hugging Face account/token if uploading datasets to the Hub

Install the LeRobot package from this repository in editable mode:

```bash
pip install -e .
```

For the full dependency set used by this checkout:

```bash
pip install -r requirements-ubuntu.txt
```

Install the Piper SDK dependencies according to your Piper SDK distribution if
they are not already available in your Python environment.

## CAN Setup

The helper script `1_init_can.sh` maps physical USB bus locations to stable CAN
interface names:

```bash
bash 1_init_can.sh
```

By default, the script expects:

```bash
can_leader:1000000
can_follower:1000000
```

Before running on a different machine, edit the `USB_PORTS` table in
`1_init_can.sh` so that each physical USB port maps to the correct interface
name and bitrate. The script loads `gs_usb`, configures bitrate, brings CAN
interfaces up, and renames them to the configured names.

You can inspect the detected CAN interfaces with:

```bash
ip -br link show type can
```

## Teleoperation

After the CAN interfaces are configured, run the provided teleoperation example:

```bash
bash 2_teleop.sh
```

The expanded command uses:

```bash
lerobot-teleoperate \
  --robot.type=piper_follower \
  --robot.port=can_follower \
  --robot.id=follower \
  --teleop.type=piper_leader \
  --teleop.port=can_leader \
  --teleop.id=leader \
  --display_data=true
```

The example script also configures two OpenCV cameras. Update `/dev/video*`,
resolution, and FPS values for your camera setup.

## Recording Datasets

Dual-port leader-follower recording:

```bash
bash 3_doubleport_record.sh
```

Single-port/direct recording:

```bash
bash 4_singleport_record.sh
```

HIL joint-delta recording with a policy checkpoint:

```bash
bash 5_HIL_record.sh
```

Before recording, update the example scripts for your environment:

- `--dataset.repo_id=your_HF_id/your_repo_id`
- `--dataset.single_task="..."`
- camera paths such as `/dev/video4`
- episode length, FPS, and reset timing
- `--policy.path=...` for HIL recording

## Piper Device Behavior

`PiperFollower` exposes seven action/observation motor features:

```text
joint1.pos
joint2.pos
joint3.pos
joint4.pos
joint5.pos
joint6.pos
gripper.pos
```

When cameras are configured, camera frames are added to the observation
dictionary using the configured camera names.

`PiperMotorsBus` normalizes Piper SDK joint values into LeRobot ranges:

- Arm joints can use `RANGE_M100_100` or degrees, depending on
  `--robot.use_degrees`.
- The gripper uses `RANGE_0_100`.
- Built-in calibration ranges are currently defined in the Piper follower and
  leader implementations.

## Safety Notes

- Verify CAN interface names before enabling the robot.
- Keep the robot workspace clear before running teleoperation or recording.
- Confirm camera indices and dataset destinations before starting long sessions.
- Use `--robot.max_relative_target` when you want LeRobot to clamp action jumps
  between the requested target and current follower position.
- The current Piper calibration load/save hooks are placeholders; fixed
  calibration ranges are defined in code.

## Relationship to Upstream Projects

This repository keeps the LeRobot package structure and version target from
Hugging Face LeRobot 0.4.3, while adding and adapting Piper support.

Piper-specific code and behavior were developed with reference to:

- [WeGo-Robotics/lerobot_piper](https://github.com/WeGo-Robotics/lerobot_piper.git)

General LeRobot functionality, dataset tooling, policies, and CLI conventions
come from:

- [huggingface/lerobot](https://github.com/huggingface/lerobot)

See `LICENSE` and source file headers for licensing and attribution details.
