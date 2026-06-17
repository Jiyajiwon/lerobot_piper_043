# LeRobot Piper for LeRobot 0.4.3

This repository adds AgileX Piper support to LeRobot 0.4.3.

It is based on the Piper integration work from
[WeGo-Robotics/lerobot_piper](https://github.com/WeGo-Robotics/lerobot_piper.git),
which targeted LeRobot 0.3.3. The Piper pieces have been refactored here to fit
the LeRobot 0.4.3 robot/teleoperator config structure, factory utilities, and
CLI workflow.

The main workflow is simple: connect Piper arms over CAN bus, use one arm as the
leader, use the other as the follower, and record LeRobot datasets from the
follower state, actions, and cameras.

## What Is Included

- `PiperMotorsBus`, a Piper SDK-backed motor bus implementation
- `piper_follower`, a LeRobot `Robot` type for the follower arm
- `piper_leader`, a LeRobot `Teleoperator` type for the leader arm
- Factory registration so the Piper devices can be created from CLI arguments
- CAN setup scripts for stable leader/follower interface names
- Example scripts for teleoperation, dataset recording, and HIL recording

Most of the repository remains the upstream Hugging Face LeRobot 0.4.3 codebase.
The Piper-specific additions live in a small number of files listed below.

## Key Files

```text
src/lerobot/motors/piper/
  piper.py                          PiperMotorsBus wrapper around the Piper SDK
  tables.py                         Piper motor model and initialization values

src/lerobot/robots/piper_follower/
  config_piper_follower.py          LeRobot robot config registration
  piper_follower.py                 Follower arm, cameras, observations, actions

src/lerobot/teleoperators/piper_leader/
  config_piper_leader.py            LeRobot teleoperator config registration
  piper_leader.py                   Leader arm action source

src/lerobot/scripts/
  lerobot_record_singleport.py      Single-port/direct recording flow
  lerobot_record_HIL_joint_delta.py HIL recording flow with a policy in the loop

1_init_can.sh                       CAN interface setup and renaming helper
2_teleop.sh                         Leader-follower teleoperation example
3_doubleport_record.sh              Dual-port dataset recording example
4_singleport_record.sh              Single-port/direct recording example
5_HIL_record.sh                     HIL joint-delta recording example
```

## Refactor Notes

The original reference implementation was written for a LeRobot 0.3.3-style
codebase. In this checkout, the Piper integration has been updated for LeRobot
0.4.3:

- `PiperFollowerConfig` registers through `RobotConfig.register_subclass`.
- `PiperLeaderConfig` registers through `TeleoperatorConfig.register_subclass`.
- `PiperFollower` follows the current `Robot` interface for observations,
  actions, camera configs, and optional action limiting.
- `PiperLeader` follows the current `Teleoperator` interface and reads leader-arm
  control values as LeRobot actions.
- `PiperMotorsBus` translates Piper SDK reads and writes into LeRobot motor bus
  behavior, including normalized joint and gripper values.
- The factory helpers recognize `--robot.type=piper_follower` and
  `--teleop.type=piper_leader`.

## Requirements

- Python 3.10 or newer
- AgileX Piper arm hardware
- CAN adapters for the leader/follower examples
- Linux SocketCAN tools for the included `1_init_can.sh` helper
- Piper SDK Python packages:
  - `piper_sdk`
  - `wego_piper`
- Camera devices if recording image observations
- A Hugging Face account/token if uploading datasets to the Hub

Install this checkout in editable mode:

```bash
pip install -e .
```

For the full dependency set used by this checkout:

```bash
pip install -r requirements-ubuntu.txt
```

If `piper_sdk` or `wego_piper` is missing, install the Piper SDK dependencies
from your Piper SDK distribution.

## CAN Setup

`1_init_can.sh` is a Linux SocketCAN helper. It maps physical USB bus locations
to stable CAN names so the rest of the scripts can refer to predictable ports:

```bash
bash 1_init_can.sh
```

The default names are:

```bash
can_leader:1000000
can_follower:1000000
```

Before running it on a new machine, edit the `USB_PORTS` table in
`1_init_can.sh`. Each physical USB port should map to the interface name and
bitrate you want to use.

You can inspect detected CAN interfaces with:

```bash
ip -br link show type can
```

## Teleoperation

After the CAN interfaces are ready, start leader-follower teleoperation:

```bash
bash 2_teleop.sh
```

The core command is:

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

This repository includes three recording entry points. They share the same basic
LeRobot dataset flow, but they are meant for different hardware/control setups.

### Dual-port leader-follower recording

```bash
bash 3_doubleport_record.sh
```

Use this for the standard two-arm setup. The leader arm is connected through
`can_leader`, the follower arm through `can_follower`, and LeRobot records the
follower observations, commanded actions, and camera frames as a dataset. This is
the usual choice when collecting demonstrations by physically moving the leader
arm.

### Single-port/direct recording

```bash
bash 4_singleport_record.sh
```

Use this when you want to record through a single follower-side CAN interface
without adding a separate `piper_leader` teleoperator to the command. The script
uses the custom `lerobot_record_singleport.py` path with `--direct_record=true`,
which is useful for simpler one-arm recording or quick checks where the full
leader-follower setup is not needed.

### HIL joint-delta recording with a policy checkpoint

```bash
bash 5_HIL_record.sh
```

Use this for human-in-the-loop recording with a trained policy loaded from
`--policy.path`. The follower, leader, cameras, and policy are all part of the
session: the policy can propose actions while the human operator can guide or
correct behavior through the leader arm. The resulting episodes can be used to
inspect, improve, or extend policy behavior on the real Piper setup.

Before recording, update the example scripts for your setup:

- `--dataset.repo_id=your_HF_id/your_repo_id`
- `--dataset.single_task="..."`
- camera paths such as `/dev/video4`
- episode length, FPS, and reset timing
- `--policy.path=...` for HIL recording

## Piper Device Behavior

`PiperFollower` exposes seven motor features as both observations and actions:

```text
joint1.pos
joint2.pos
joint3.pos
joint4.pos
joint5.pos
joint6.pos
gripper.pos
```

When cameras are configured, their frames are added to the observation dictionary
under the configured camera names.

`PiperMotorsBus` normalizes Piper SDK joint values into LeRobot ranges:

- Arm joints can use `RANGE_M100_100` or degrees, depending on
  `--robot.use_degrees`.
- The gripper uses `RANGE_0_100`.
- Built-in calibration ranges are currently defined in the Piper follower and
  leader implementations.

## Safety Notes

- Check CAN interface names before enabling the robot.
- Keep the robot workspace clear before teleoperation or recording.
- Confirm camera indices and dataset destinations before long sessions.
- Use `--robot.max_relative_target` if you want LeRobot to clamp action jumps
  between the requested target and current follower position.
- The current Piper calibration load/save hooks are placeholders; fixed
  calibration ranges are defined in code.

## Related Projects

- [WeGo-Robotics/lerobot_piper](https://github.com/WeGo-Robotics/lerobot_piper.git):
  Piper support for LeRobot 0.3.3, used as the main reference for this refactor.
- [agilexrobotics/piper_sdk](https://github.com/agilexrobotics/piper_sdk): the
  official Piper robot arm SDK used by the Piper motor bus layer.
- [huggingface/lerobot](https://github.com/huggingface/lerobot): the upstream
  LeRobot project that provides the dataset, training, policy, and CLI tooling.

See `LICENSE` and source file headers for licensing and attribution details.
