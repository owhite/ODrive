## The ODrive Homing and Endpoint Branch

The default ODrive firmware relies on your encoder to find a reference position, by issuing the command:

```
<odrv>.<axis>.requested_state = AXIS_STATE_ENCODER_INDEX_SEARCH
```

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
Each axis has two endstops: a "min" and a "max".  Each endstop has the following properties:

Name |  Type | Default
--- | -- | -- 
gpio_num | int | 0
enabled | boolean | False
offset | int | 0
debounce_ms | float | 100.0
is_active_high | boolean | False

### gpio_num
The GPIO pin number, according to the silkscreen labels on ODrive. Set with these commands:
```
<odrv>.<axis>.max_endstop.config.gpio_num = <1, 2, 3, 4, 5, 6, 7, 8>
<odrv>.<axis>.min_endstop.config.gpio_num = <1, 2, 3, 4, 5, 6, 7, 8>
```

### enabled
Enables/disables detection of the endstop.  If disabled, homing and e-stop cannot take place. Set with:
```
<odrv>.<axis>.max_endstop.config.enabled = <True, False>
<odrv>.<axis>.min_endstop.config.enabled = <True, False>
```

### offset
This is the location along the axis, in counts, that the endstop is positioned at.  For example, if you want a position command of `0` to represent a position 100 counts away from the endstop, the offset would be -100 (because the endstop is at position -100). Set with:
```
<odrv>.<axis>.max_endstop.config.offset = <int>
<odrv>.<axis>.min_endstop.config.offset = <int>
```

### debounce_ms
The debouncing time, in milliseconds, for this endstop.  Most switches exhibit some sort of bounce, and this setting will help prevent the switch from triggering repeatedly. It works for both HIGH and LOW transitions, regardless of the setting of `is_active_high`. Debouncing is a good practice for digital inputs, read up on it [here](https://en.wikipedia.org/wiki/Switch). 'debounce_ms' units are in miliseconds. Set it with:

```
<odrv>.<axis>.max_endstop.config.debounce_ms = <Float>
<odrv>.<axis>.min_endstop.config.debounce_ms = <Float>
```

### is_active_high
This is how you configure the endstop to be either "NPN" or "PNP".  An "NPN" configuration would be `is_active_high = False` whereas a PNP configuration is `is_active_high = True`.  Refer to the following table for more information:

![Endstop configuration](https://github.com/owhite/ODrive/blob/master/docs/Endstop_configuration.png)  
3D printer endstops (like those that come with a RAMPS 1.4) are typically configuration **4**.

### Configuring an endstop

You can access these configuration properties through odrivetool.  For example, if we want to configure a 3D printer-style minimum endstop on GPIO 5 for homing, and you want your motor to pull off the endstop about a quarter turn with a 8192 cpr encoder, you would set:

```
<odrv>.<axis>.min_endstop.config.gpio_num = 5
<odrv>.<axis>.min_endstop.config.is_active_high = 4
<odrv>.<axis>.min_endstop.config.enabled = True

<odrv>.save_configuration()
<odrv>.reboot()
```

other config items that might help are:
```
<odrv>.<axis>.encoder.config.use_index = True
<odrv>.<axis>.controller.config.homing_speed
<odrv>.<axis>.min_endstop.config.offset = -2048;
<odrv>.<axis>.controller.config.vel_limit = 50000
```

`use_index` will be needed for encoders that still require an index search. (e.g., ABI does, SPI does not). 

`homing_speed` is in counts/second. Default is 2000. It refers to the travel speed during movement to the min_endstop.

`offset` works as a home switch position. It applies to min_endstop and represents the number of counts required to get home. Suppose home is 100 counts away from min_endstop, set this value to -100. 

*WETMELON PLEASE CONFIRM if the statement about vel_limit is true!!!*
`vel_limit` is the speed in counts/second and is the travel speed to reach offset. 

### Additional endstop devices

In addition to phyiscal switches there are other options for wiring up your endpoint detectors - you will have to work out the details of connecting your device but here are some suggested approaches:

![Endpoint figure](https://github.com/owhite/ODrive/blob/master/docs/endpoint_figure.png)

### Testing
Once motor and encoder calibration is complete, and your endpoint detectors are connected to the gpio pins, power up your ODrive. You can test your endstop devices. Try activating your endpoint devices and test the states of these variables:
```
<odrv>.<axis>.max_endstop.endstop_state
<odrv>.<axis>.min_endstop.endstop_state
```

Give it a try. Click your switches, or put a magnet on your hall switch and see if the states change. 

After testing, as always:
* `odrv0.save_configuration()` because everything is working, always save
* `odrv0.reboot()`and then bounce the odrive

### Calibration is still required
For homing, your motor/odrive configuration still needs to be calibrated. Remove all load on your motor, it has to be able to spin freely. Come out of reboot and run:

`<odrv>.<axis>.requested_state = AXIS_STATE_FULL_CALIBRATION_SEQUENCE`

the motor should beep and complete it's calibration. The calibration sequence should slowly rotate in one direction, and slowly return back. Store the fact that you've succeeded with:  
```
<odrv>.<axis>.motor.config.pre_calibrated  = True
<odrv>.<axis>.encoder.config.pre_calibrated = True
```
Then save_configuration and reboot.

### Performing a homing sequence. 

Once everything is ready, this is an example sequence of commands you can use for manually performing homing. You probably need to do something more elaborate for when you build the next consciously self-aware android, but try manual homing first with: 

```
<odrv>.<axis>.requested_state = AXIS_STATE_ENCODER_INDEX_SEARCH
<odrv>.<axis>.requested_state = AXIS_STATE_CLOSED_LOOP_CONTROL
<odrv>.<axis>.requested_state = AXIS_STATE_HOMING
```

`INDEX_SEARCH` is required for some encoders  
`CLOSED_LOOP_CONTROL` arms the motor  
`AXIS_STATE_HOMING` initiates the homing. 

*NOTE:* sometimes odrivetool will not have heard of `AXIS_STATE_HOMING` and in that case you can run:
* `<axis>.requested_state = 11`

#### WETMELON !!! THE MAIN PROBLEM HERE IS THIS SECTION AND MY SECTION JUST ABOVE THIS ARE KIND OF DIFFERENT: 
## Homing
Homing is possible once the ODrive has closed loop control over the axis.  To trigger homing, we use must first be in AXIS_STATE_CLOSED_LOOP_CONTROL, then we call`<odrv>.<axis>.controller.home_axis()`  This starts the homing sequence.  The homing sequence works as follows:

1. Verify that the `min_endstop` is `enabled`
2. Drive towards the `min_endstop` in velocity control mode at `controller.config.homing_speed`
3. When the `min_endstop` is pressed, set the current position = `min_endstop.config.offset`
4. Request position control mode, and move to the positon = `0`

### Homing at startup
It is possible to configure the odrive to enter homing immediately at startup. For safety reasons, we require the user to specifically enable closed loop control at startup, even if homing is requested.  Thus, to enable homing at startup, the following must be configured:

```
<odrv>.<axis>.config.startup_closed_loop_control = True
<odrv>.<axis>.config.startup_homing = True
```


### Notes on how the code works. 
#### _ WETMELON PLEASE CONFIRM !!! : 
For background, this branch initiates the homing process in the function `axis::run_state_machine_loop()`. When the user calls `AXIS_STATE_HOMING` the program is sent to `controller::home_axis()`. That function sets up the speed of travel, and sets the axis state to `HOMING_STATE_HOMING`. `Axis::run_closed_loop_control_loop` handles operations when the motor is armed. This function tests that `HOMING_STATE_HOMING` state is set, if it is, it runs the motor until it reaches `set_pos_setpoint`. After that occurs, the state `HOMING_STATE_MOVE_TO_ZERO` is then set, and the motor to travels to `config.offset`. How simple was that? 
