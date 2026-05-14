---
noteId: "5fa927604f8211f194a2c3b1eecd91b7"
tags: []

---

# Simulation World & Vehicle Model

## Overview

The Gazebo side defines **where the drone flies**, **what it flies**, and
the **physics regime**. The world file
(`bt_gazebo/worlds/betaloop_iris_betaflight_demo_harmonic.sdf`) hosts a
3DR-style iris quadrotor (`betaloop_iris_with_standoffs`), a red 1m³ box
as the visual target, a ground plane, and a sun. The world is geo-located
near Pittsburgh so the plugin's virtual-GPS branch produces realistic
lat/lon. Two demo models add a 2-DoF gimbal and a debug-marker variant.

## Key Decisions

- **World frame is ENU** with WGS84 spherical coordinates
  (`...harmonic.sdf:5-12`). The plugin converts ENU → NED for the FDM
  packet (`BetaflightPlugin.cc:756`, `:804`) but switches to a
  geographic frame (lon/lat/alt) when `<spherical_coordinates>` is
  present, enabling Betaflight's virtual-GPS mode.
- **Physics tick is 4 ms** (`max_step_size=0.004`, `real_time_factor=1`,
  `...harmonic.sdf:14-18`) → 250 Hz sim, comfortably faster than the
  50 Hz autopilot loop and Betaflight's PID rate.
- **Iris with QuadX motor layout** —
  `betaloop_iris_with_standoffs/model.sdf:734-790` documents and pins the
  mapping between Betaflight's `motorSpeed[i]` slots and physical rotor
  joints. This is the load-bearing decision for "does it fly stably".
- **Default IMU sensor is `iris/imu_link::imu_sensor`** — the plugin
  searches by scoped name (`BetaflightPlugin.cc:521-544`); changing the
  link name in the model **must** be mirrored in the plugin tag.
- **Required system plugins are listed explicitly** — physics, sensors
  (Ogre2), user-commands, scene-broadcaster, IMU (`...harmonic.sdf:21-31`).
- Sun + 5000×5000 ground plane only — no other obstacles. Visual tracker
  only needs the red box.

## Constraints

- Iris model is QuadX with four rotors only. `MAX_MOTORS=255` in the
  plugin (`BetaflightPlugin.cc:69`) is purely a packet limit, not a
  physical capability.
- Each rotor's `<turningDirection>` (`cw`/`ccw`) flips the sign of torque
  applied. Getting any of the four wrong inverts that motor and the
  drone will not lift off — there is **no protection** in software.
- `<spherical_coordinates>` block is required for virtual GPS. Without it,
  position is reported in NED metres relative to spawn instead of
  lat/lon (`BetaflightPlugin.cc:813-834`).
- The model includes inertia, mass, and `<collision>` geometry tuned for
  the Iris; substituting a different airframe needs all of those plus
  the motor↔BF-slot mapping recomputed.

## Open Questions

- Aerodynamics: there is no LiftDragPlugin in the world or in the iris
  model — motor force is the only aerodynamic force modeled, so prop
  wash, ground effect, and translational drag are absent. Accuracy of
  tracking near hover is unverified.
- The demo variant (`betaloop_iris_with_standoffs_demo`) adds a 2-DoF
  gimbal but is not used by the live world — keep, drop, or merge?
- Wind / disturbances: nothing models them. Robustness of the visual
  controller under perturbation is untested.
