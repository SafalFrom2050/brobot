# Brobot

Brobot is a plan to combine the Freenove 4WD Car Kit for Raspberry Pi Pico W
with the existing `pico-w-room-controller` smart-home firmware.

The Pico W will remain the real-time body controller for motors, infrared
devices, safety, telemetry, lights, and robot emotions. A companion computer
can be added later for camera processing, planning, and AI without giving it
unrestricted access to the hardware.

The first milestone is intentionally smaller: safe manual vehicle control over
the local network, with a browser joystick, a command timeout, battery
telemetry, and an emergency stop.

See [the project plan](docs/PROJECT_PLAN.md) for the proposed architecture,
protocol, safety requirements, hardware mapping, roadmap, and open decisions.

## Source projects

- [Freenove 4WD Car Kit for Raspberry Pi Pico](https://github.com/Freenove/Freenove_4WD_Car_Kit_for_Raspberry_Pi_Pico)
- [pico-w-room-controller](https://github.com/SafalFrom2050/pico-w-room-controller)

## Status

Planning. No robot firmware has been imported or implemented in this
repository yet.
