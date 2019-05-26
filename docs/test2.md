## The ODrive Homing and Endpoint Branch

The default ODrive firmware relies on your encoder to find a reference position, by issuing the command:

* `<odrv>.<axis>.requested_state = AXIS_STATE_ENCODER_INDEX_SEARCH`

In this scenario the motor will spin until the Z pin changes state. That's fine to tell you the rotational position of the motor. But what happens when your motor has to rotate multiple times before your board is aware that it is in the right position? 

For example, many robotics and CNC activities require a procedure known as _homing_. In these situations it is useful to allow your motor to move until a physical or electronic device orders the system to stop. That _endstop_ can be used as a reference point, and once the ODrive has hit that position it may then want to move to a final home position.
