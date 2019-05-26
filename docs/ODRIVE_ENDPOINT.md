## The ODrive Homing and Endpoint Branch

The default ODrive firmware relies on your encoder to find a reference position, by issuing the command:

* `<odrv>.<axis>.requested_state = AXIS_STATE_ENCODER_INDEX_SEARCH`

In this scenario the motor will spin until the Z pin changes state. That's fine to tell you the rotational position of the motor. But what happens when your motor has to rotate multiple times before your board is aware that it is in the right position? 

For example, many robotics and CNC activities require a procedure known as _homing_. In these situations it is useful to allow your motor to move until a physical or electronic device orders the system to stop. That _endstop_ can be used as a reference point, and once the ODrive has hit that position it may then want to move to a final home position.

This version of ODrive firmware enables users to use the ODrive gpio pins to connect to phyiscal limit switches, or other sensors, that can serve as endpoint detectors. 

### Getting started

Before you plunge into using this endpoint firmware, you will have to perform the following steps. 

* Checkout the ODrive-Endpooint firmware
* Compile and load the firmware
* Configure your ODrive to calibrate a motor
* Perform encoder offset calibration
* Save the configuration and reboot

Describing how to complete above steps is outside of the scope of this documentation. Refer to [this site](https://github.com/madcowswe/ODrive) for more information, and don't hesitate to visit the [odrive forum](https://discourse.odriverobotics.com/) or [odrive discord chat site](https://discourse.odriverobotics.com/t/come-chat-with-us/281). There is a lot of good information at those resources. 

### Configuring the ODrive to perform homing. 

First, you will  need to be able to wire some devices to the ODrive digital inputs. There are a lot of options for wiring up your endpoint detectors - you will have to work out the details of connecting your device. To get you started this is a diagram of switches and devices that could work to detect your endpoints. 

![Endpoint figure](/endpoint_figure.png)

Once motor and encoder calibration is complete, and your endpoint detectors are connected to the gpio pins, power up your ODrive. You will then need to set these variables:

* `<odrv>.<axis>.max_endstop.config.gpio_num = <1, 2, 3, 4, 5, 6, 7, 8>` pick one
* `<odrv>.<axis>.min_endstop.config.gpio_num = <1, 2, 3, 4, 5, 6, 7, 8>` pick one
* `<odrv>.<axis>.max_endstop.config.enabled = <True, False>` probably want True

`enabled` should be self explanatory
`gpio_num` refers to the pin used to detect a change in your switch. When you are selecting GPIO pins make sure they do not conflict with the default pins for step, dir or UART 

Now you can test your endstop devices. Once you are at this point you should be able to use your devices to change the states of these variables:
* `odrv0.axis0.max_endstop.endstop_state`
* `odrv0.axis0.min_endstop.endstop_state`

Give it a try. Click your switches, or put a magnet on your hall switch and see if the states change. If that works for a more final configuration, you can consider changing the following variables:
* `<odrv>.<axis>.max_endstop.config.is_active_high = <True, False>` 
* `<odrv>.<axis>.max_endstop.config.debounce_ms = <Float>` 
* `<odrv>.<axis>.min_endstop.config.is_active_high = <True, False>` 
* `<odrv>.<axis>.min_endstop.config.debounce_ms = <Float>` 
* `<odrv>.<axis>.min_endstop.config.offset = <int>` 
* `<axis>.controller.config.vel_limit = 50000`
* `<axis>.encoder.config.use_index = True` 

`is_active_high` sets the polarity of the endstop. When the switch is pressed it pulls the pin LOW (this is very common for endstop switches), you want to set this to “False”.

`debounce_ms` is a good practice for digital inputs, read up on it [here](https://en.wikipedia.org/wiki/Switch). 'debounce_ms' units are in miliseconds. 

`offset` works aas a home switch position. It applies to min_endstop and represents the number of counts required to get home. Suppose home is 100 counts away from min_endstop, set this value to -100. The speed it will move to the offset is set by `vel_limit`. *WETMELON PLEASE CONFIRM if the statement about vel_limit is true!!!*

`use_index` will be needed for encoders that still require an index search. (e.g., ABI does, SPI does not). 

* `odrv0.save_configuration()` because everything is working, always save
* `odrv0.reboot()`and then bounce the odrive

After setting up your configuration, always make sure things are getting stored by running:
* `<odrv>.save_configuration()`
and then reboot. 

### Calibration is still required

For homing, your motor/odrive configuration still needs to be calibrated. Remove all load on your motor, it has to be able to spin freely. Come out of reboot and run:

`<axis>.requested_state = AXIS_STATE_FULL_CALIBRATION_SEQUENCE`

the motor should beep and complete it's calibration. The calibration sequence should slowly rotate in one direction, and slowly return back. Store the fact that you've succeeded with:
`<axis>.motor.config.pre_calibrated  = True 
`<axis>.encoder.config.pre_calibrated = True
Then save_configuration and reboot.

### Performing a homing sequence. 

Once everything is ready, this is an example sequence of commands you can use for manually performing homing. You probably need to do something more elaborate for when you build the next consciously self-aware android, but try manual homingfirst with: 

* `<axis>.requested_state = AXIS_STATE_ENCODER_INDEX_SEARCH`
* `<axis>.requested_state = AXIS_STATE_CLOSED_LOOP_CONTROL`
* `<axis>.requested_state = AXIS_STATE_HOMING`

`INDEX_SEARCH` is required for some encoders
`CLOSED_LOOP_CONTROL` arms the motor
`AXIS_STATE_HOMING` initiates the homing. 

*NOTE:* sometimes odrivetool will not have heard of `AXIS_STATE_HOMING` and in that case you can run:
* `<axis>.requested_state = 11`

### Notes on how the code works. 

#### _ WETMELON PLEASE CONFIRM !!! : 
For background, this branch initiates the homing process in the function `axis::run_state_machine_loop()`. When the user calls `AXIS_STATE_HOMING` the program is sent to `controller::home_axis()`. That function sets up the speed of travel, and sets the axis state to `HOMING_STATE_HOMING`. `Axis::run_closed_loop_control_loop` handles operations when the motor is armed. This function tests that `HOMING_STATE_HOMING` state is set, if it is, it runs the motor until it reaches `set_pos_setpoint`. After that occurs, the state `HOMING_STATE_MOVE_TO_ZERO` is then set, and the motor to travels to `config.offset`. How simple was that? 

