---
noteId: "703489304f8211f194a2c3b1eecd91b7"
tags: []

---

# Sim ↔ SITL Bridge Plugin

## Overview

`BetaflightPlugin.cc` is the **physics bridge**: a Gazebo Harmonic system
plugin (gz-sim8 / gz-plugin2) that on every `PreUpdate` (a) reads motor
PWM packets from Betaflight on UDP 9002, (b) translates them into joint
torques applied to each rotor link via PID velocity controllers, and (c)
sends back the simulated FDM state (IMU, pose, velocity, ESC RPM) on UDP
9003. It is the only Gazebo→FC integration point. The plugin is loaded
inside the iris model SDF, **not** the world SDF, so re-use with a
different airframe means re-attaching the plugin.

## Key Decisions

- **UDP datagrams, not gz-transport.** The bridge bypasses Gazebo's pub/sub
  layer in favor of raw UDP so the same wire protocol works with PX4 /
  ArduCopter SITL conventions. Packet shapes mirror ArduPilot's `SIM_*`
  family.
- **`betaflightOnline` gating.** Until the FC sends its first valid PWM
  packet, Gazebo runs free with no motor force (`BetaflightPlugin.cc:582-587`).
  This avoids the chicken-and-egg of "sim must run for SITL to compute,
  SITL must compute for sim to run".
- **Adaptive recv timeout.** 1 ms while offline (drop fast), 1 s once
  online (tolerate jitter), `connectionTimeoutMaxCount` consecutive misses
  before declaring offline (`:646-687`).
- **Per-rotor velocity PID converts BF motor command → joint torque**
  (`:606-632`). `cmd = motorSpeed[i] * maxRpm (838 rad/s)`, then PID drives
  the joint to that velocity via `JointForceCmd`. `<turningDirection>`
  flips the sign of the velocity target so torque reaction matches.
- **NED frame for FDM, ENU for virtual GPS.** Gazebo world is ENU, but
  Betaflight expects NED IMU/attitude. The plugin applies a 180° roll
  (`gazeboToNED`) when filling `imuOrientation` / `velocity` / `position`,
  then switches to lon/lat/alt + raw ENU velocity when a
  `SphericalCoordinates` component is present on the world
  (`:756-834`).
- **Per-rotor SDF parameters.** PID gains, max/min cmd, `jointName`,
  `turningDirection`, `rotorVelocitySlowdownSim`, optional filter cutoff
  (`getSdfParam` helper at `:84-103`).
- **ESC sensor is partly faked.** `escTemperature/Voltage/Current/Consumption`
  are zero-initialized; only `escRpm[i]` is filled from joint velocity
  (`:836-856`).

## Constraints

- Bind address is hardcoded `127.0.0.1:9002` and send target is
  `127.0.0.1:9003` (`:329`, `:859`). Multi-host or multi-vehicle (more
  than one plugin instance) is impossible without code changes.
- `SO_REUSEADDR` + `O_NONBLOCK` are set, but only one plugin can bind
  9002 per host.
- `ServoPacket` is `float motorSpeed[255]` — fixed-size struct; partial
  recvs of `recvSize < expectedPktSize = sizeof(float)*rotors.size()` are
  discarded.
- `fdmPacket` includes `escTemperature/Voltage/Current/Consumption` sized
  `[4]` — implicitly assumes ≤4 rotors for ESC telemetry even though
  `MAX_MOTORS=255`. Larger configurations would write OOB.
- IMU search is by substring match on the sensor name (`:533-544`) — name
  collisions across multiple Imu components would pick the wrong one.

## Open Questions

- The plugin's PID gains are pulled from SDF but the iris model sets
  `vel_p_gain=0.01` with all others zero — is this hand-tuned, or default
  copy-paste? Affects motor responsiveness.
- `connectionTimeoutMaxCount` is `5` in the iris SDF but `10` is the
  plugin default. Which is correct?
- `rotorVelocitySlowdownSim` is set to `1` everywhere in the iris model,
  but the plugin defaults to `10.0` — what is the physical intent?
- No documented contract for FDM packet timestamp source (sim time vs.
  wall) — currently sim time, but consumers must agree.
