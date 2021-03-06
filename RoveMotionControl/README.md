# Rove Joint Control Framework

## Overview
Programmers: Drue Satterfield, David Strickland

Used for controlling motor controllers/drivers and easily manipulating joints and other axises of motion. 
The basics are: there's four modules that work together to control the joints; an axis of motion manager class to oversee the other two and general operation, an algorithm class to control any special controls features, an output class to control the hardware layer at the bottom, and a FeedbackDevice class that represents sensors, generally for giving information to the algorithm class. The user interacts with the axis interface for main motion manipulation and management, while individual classes can be called to modify settings of operation specfically associated with them.
Those four abstract classes set up the general framework, and what each classes' general respondibilities are. The specific classes derive from the abstracts, and implement carrying out those responsibilities.
Each module is expected to take a specific, standarized kind of input/output data; because the types are standardized, different modules can work with each other without having to know specifically what said module is doing, trusting that they'll be able to handle their own responsibilities independently. 
There's an input/output type for speed data, for position data, for power data, etc. Which type each module expects just depends on what the actual implementation is.
[More Details Here](https://github.com/MST-MRDT/ArmBoardSoftware/wiki/Joint-control-framework-overview)

## Files
The `AbstractFramework.h` and `AbstractFramework.cpp` files holds the top level abstract classes that form the backbone of the framework. Enums, constants, and macros are kept in `RoveJointUtilities.h` file. `RoveJointControl.h` is the primary external include for the framework. The rest of the files define subclasses of the abstract base classes. 

## Dependencies
* RoveBoard
* Other files in roveware such as RoveDynamixel

## Usage
* 0) Include `RoveMotionControl.h`. Then, include the derivative classes you want to use.
* 1) Construct the `OutputDevice` representing the device (motor controller, h-bridge, etc) used to move the axis of motion.
* 2) If the axis is closed-loop controlled, then also construct a `FeedbackDevice` representing the device on the axis and the `IOAlgorithm` representing the desired closed-loop algorithm. Otherwise if it uses any other kind of algorithm to interpret commands for the motor, construct the `IOAlgorithm` without a feedback device. Using no `IOAlgorithm` is also an option, if the `OutputDevice` can understand the commands on its own.
** Note that if you want to use more complicated control schemes, you can weave multiple `IOAlgorithm`'s together; `IOAlgorithm` supports lacing multiple algorithms in order so that multiple are ran in sequence whenever the axis is updated. More details in its module description below
* 3) Finally, construct the `MotionAxis` interface that represents this axis or joint by passing in the `OutputDevice` and the `IOAlgorithm`. The constructor will also require what kind of values the `MotionAxis` should expect, such as positional values or speed values. This is so that the interface knows how to properly interpret commands.
* 4) The user should now be able to control the axis using the `MotionAxis` object.

## Modules
### Motion Axises
Interface for controlling the overall joint or axis of motion from the main program's perspective. It handles all duties of controlling the axis.
* `SingleMotorAxis` Axis controlled by a singular motor device
* `DifferentialAxis` Mechanical differential joint (two motors attached, with both motors technically controlling two degrees of freedom at once; making them move together causes the joint to move up/down so to speak, making them move in opposite causes the joint to spin in place). Typically, two instances are used together to represent the two degrees of motion the mechanical joint can do, with the user explicitely tying them together via api

### IOConverters
Algorithms that convert the input from base station to whatever is needed for the output device interpret the command.

**IMPORTANT:**  Note that these algorithms, when they use feedback such as PID loops, typically need to be periodically called, with the timing done externally, until `MotionAxis` returns an `OutputComplete` status, as each call only executes the control loop once instead of waiting until completion.

* `PIAlgorithm` Closed loop algorithm, using PI logic. Logic is generalized, PI constants are accepted through constructors. To be used when position is received from the base station and the speed is to be sent to the device, which in turn, returns feedback of the device's current location.
* `TtoPPOpenLConverter` Open Loop algorithm that's used to convert torque to power percent values. It does this mathematically, checking what type of motor is being used and using that information to convert torque to voltage. The class also is told or finds out via sensor what the voltage is for the motor and uses that to convert the desired voltage values into power percent.

### Output Devices
Controls the devices which move the arm, such as motor controllers, using the hardware specifics of the devices, such as what GPIO pins.
* `DirectDiscreteHBridge` An h-bridge made out of discrete components (not an IC), and the microcontroller lines directly control it (no other devices in between them). Two inputs, NTransistor1 and NTransistor2. It's assumed the other two transistors will be p-type transistors so they don't need control lines to the microcontroller. 
* `Sdc2130` DEPRECATED, TESTED BUT THE WAY WE USE IT HISTORICALLY IS CRAP AND NO ONE WANTS TO UPDATE IT. A brushed DC motor controller, capable of being controlled via PWM or serial, and controlling the motor based on speed, position, or torque with feedback capabilities. Currently the only implemented modes are controlling it via PWM and moving by taking in a speed position. (Best we got it working is to make it slightly increment position when we tell it to move. So the only set up movement is for the device to take in a speed value, look at the value and if it's greater than 0, slightly move it up, and if it's less than 0 slightly move it down.)
* `DynamixelController` DEPRECATED, BUGGY AND UNTESTED. Interfaces with either MX or AX dynamixel. They can operate in Wheel, Joint, and Multi-rotation modes which take speed, position, and position respectively. Requires `RoveDynamixel.h`.
* `GenPwmPHaseHBridge` A class that represents any case where the h-bridge is controlled via two pins, specifically a speed pin controlled by PWM and a direction/phase pin. This class also has a great deal of extended functionality built in, due to being the 
primary class used historically so the one with the most development. For more details, see its h file.
* `RCContinuousServo` Generic class for any RC Continuous Servo device.
* `VNH5019` VNH5019 H-bridge IC
* `VNH5019WithPCA9685` An odd output device that controls potentially several VNH5019 h-bridges at once, with a PCA9685 pwm driver, which can output several pwm drivers. The program tells the PCA what pwm to send the various vnh's, while the user still manually controls the different vnh directions using gpio pins. Up to 16 VN5019's can be controlled with this setup on one PCA. Each instance of this class represents one VNH that's being controlled, and each instance shares the PCA information.
*  `BTM7753GWithPCA9685` Similar to the one above, except the BTM7753 h bridges are controlled via 2 pwm channels individually and no gpio pin logic, so it's entirely moved by the pca. Each pca can control up to 8 btm's. Each instance of the class represents one btm that's being controlled, and the instances all share the pca information.

### Feedback Devices
Feedback devices are used to help determine sensory information about the axises. For example, some are used by the `IOAlgorithm` class to gain information on the system's physical state such as its current position.
* `Ma3Encoder12b` MA3 magnetic encoder, 12 bit pwm version. Communicates via PWM, 12-bit resolution of degrees over 360 degrees.

### Stopcap Mechanisms
Stopcap devices are used to signal the motion of axis that it needs to stop moving, either entirely or limited to a single direction. An example of this is 
limit switches.
* `DualLimitSwitch` A class that represents when two limit switches are used to keep an axis of motion from going too far forward or backward. One represents 
the limit of positive direction, one negative
* `SingleStopLimitSwitch` A class that represents when one limit switch is used on an axis of motion, where triggering it stops the motion entirely and 
requires removing the switch in software before it can be moved again.

### Experimental
Classes which have not been proven to work and are still in development.
* `GravityCompensation` class that tries to account for gravity during axis motion. Designed to act as a supporting IOConverter to another IOConverter rather than usually directly controlling an axis itself.
It relies on both GravityInertiaSystemStatus.h and TtoPPOpenLConverter, the former to compute the heavy gravitational math and the latter to 
convert the resulting torque into power percent. 
* `GravityInertiaSystemStatus` this class is fairly unique, it's designed to be able to compute how much torque the robotic arm is under at any given moment due to gravity and inertia. It does this by taking in mechanical constants about the arm and tracks the different feedback device's needed to figure out where the arm is
currently at. Note that this class needs its own separate call to compute its math separate from the rest of the RJC; it has an update() function that should be called periodically, this will signal to the class that it needs to update its results for how much torque the arm is currently under. 
The rest of the RMC that tries to compensate for gravity or inertia will call this class everytime they are asked to update their movements. The reason it exists separately is that the math is heavy enough to where it would be unreasonable to run it every single time the controls update, 
so it should be called periodically on a separate thread to the rest of RMC, updating independantly as fast as the user wants the math to be updated.
* `PIV Converter` A cascaded PID loop composed of two loops, a slower position loop running on the outside and a quicker velocity loop that's being fed the results of the outer. 
Requires both positional and velocity encoders
* `Velocity Deriver` A feedbackDevice class that tries to interprolate speed based off of applying a derivative to a positional sensor's data.

### Deprecated
Classes which aren't ready to use and we don't particularly plan to use anytime soon
*`DynamixelController` A class designed to operate dynamixel controllers as an output device

## Examples
* Examples can be found in the ArmBoardSoftware repo, where RoveMotionControl was created.

## Extension
In order to implement new modules into the framework:
* 1) Inherit from the proper abstract class.
* 2) Define a constructor.
   * ...If it's a device class then it should take in whatever parameters it needs to output properly such as hardware pins.
   * ...If it's a closed-loop algorithm class, then the constructor should take in any parameters used to configure the algorithm as well as internally set the input and output types. The constructor should be public since the user will need to construct them personally. As well, it should take in any FeedbackDevices it needs.
   * ...In the constructor, make sure to define what the module's expected input type is (and or output type if it wants both, check the abstract classes to see what has what).
* 3) Document the module in this README.
* 4) Make sure to test the module and see that it compiles properly
* 5) Remember, operation-wide functions should be implemented in the MotionAxis modules, while specific operation settings can be handled by the other classes.
