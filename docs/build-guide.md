# Build guide — SimpleFOC Gimbal-BLDC Balancing Robot

Skill level: comfortable with Arduino IDE/PlatformIO, soldering header jumpers and handling LiPo packs.

## 1. Print the parts

Print-ready STL files are in `source/upstream/CAD/STL/`. Editable SolidWorks parts are in `source/upstream/CAD/SolidWorks/`.

| File | Infill | Layer height | Notes |
|---|---|---|---|
| `FOC_balancer.STL` (center frame) | 30% | 0.15–0.2 mm | carries motors, encoders, battery |
| `wheel.STL` x2 | 30% | 0.10–0.15 mm | print accurately; motor bolt circle must line up |
| `arduino_holder.STL` | 30% | 0.15–0.2 mm | modular MCU mount |
| `top_plate.STL` | 20% | 0.15–0.2 mm | |

PLA works; PETG or ABS is fine. Before printing, compare your motor and encoder hole spacing with the frame
model. The original holes were drawn for an iPower GM4108 and a CUI AMT10x encoder.

## 2. Mechanical assembly

1. Fit the two encoders to the motor shafts per the AMT10x instructions and set the resolution to 500 CPR.
2. Bolt each motor into the frame with M3 x 10 mm screws.
3. Mount each wheel rim to the motor rotor, then stretch the RC tire over the rim.
4. Fit the MCU holder to the frame, then the top plate. M4 x 50 mm screws with nuts clamp the stack.
5. Seat the battery in the frame pocket; route the lead to an inline switch.

## 3. Electronics

- Stack two SimpleFOC Shields on the D1 R32. Set the shield solder jumpers so the pins match
  `balancer_pinouts.h` (ESP32 section):
  motor 1 A/B/C/EN = 27/5/16/12, motor 2 A/B/C/EN = 18/25/13/14,
  encoder 1 A/B = 26/17, encoder 2 A/B = 36/19.
- Connect each motor's three phases to its shield terminal block.
- Wire the MPU6050 to the I2C pins (SDA/SCL, 3.3 V, GND).
- Feed battery power to the shield power terminals through the switch. Do not power the motors from USB.

### Supply voltage: battery and firmware disagree upstream

The original build runs from a 3S LiPo: 11.1 V nominal, 12.6 V fully charged, about 9.9 V when empty. The
original sketch still sets `driver1.voltage_power_supply = 8;` and `driver2.voltage_power_supply = 8;`. SimpleFOC
uses that number to convert commanded volts into PWM duty, so with a full 3S pack the motors receive roughly
1.5 times the voltage the controller asks for, and the inherited 8 V limit actually lets the full pack voltage
through. Set
both values to your pack voltage (12 for a 3S pack) before the first power-up, set `motor.voltage_limit`
explicitly, and retune the PID gains, because the original gains were tuned with the 8 V setting.

## 4. Firmware

1. Install the SimpleFOC library and the ESP32 board package.
2. Open `source/upstream/Arduino/simplefoc_balancer/simplefoc_balancer.ino` from this repository.
3. On current SimpleFOC releases, replace the brace-initialized PID and low-pass filter objects with
   constructor calls, for example `PIDController pid_stb(30, 100, 1, 100000, 7);` and
   `LowPassFilter lpf_pitch_cmd(0.07);`.
4. Change `voltage_power_supply` on both drivers from 8 to your pack voltage (12 for 3S) and add an explicit
   `motor.voltage_limit` on both motors (see "Supply voltage" above).
5. Flash, open the serial monitor at 250000 baud and confirm IMU init and motor alignment.

## 5. Tuning and driving

- Hold the robot upright, power on, and let alignment finish before releasing it.
- Tune the stabilisation PID first, then the velocity PID, using the serial Commander.
- Pair a phone with Bluetooth name `FOCBalancer_esp` and use a byte-protocol joystick app (upstream used Gibalo)
  for throttle and steering.

## Troubleshooting

- Robot falls immediately in one direction: IMU orientation or sign wrong; check pitch sign in serial output.
- Motor buzzes without turning: phase or encoder wiring, wrong pole pairs (firmware uses 11), or encoder CPR mismatch.
- Brown-outs or resets: battery sag; check pack charge and wiring gauge.
- `IMU connection problem`: check SDA/SCL wiring and 3.3 V supply.

## Safety

Use a balance charger, never leave a charging LiPo unattended, and store packs at storage voltage. Keep fingers
and cables clear of the wheels when the motors are enabled.

Based on Arduino SimpleFOC Balancer by the SimpleFOC project, MIT License.
