

lerobot callibrate command
```
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




when run the lerobot teleoprate command , sometimes calibration values mismatch between leader and follower arm or no calibration file found, so the teleoperate command ask you recallibrate the arms. so I think you can callibrate the arms with teleoperate command without precallibration through lerobot callibrate command 

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