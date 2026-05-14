---
noteId: "b9d84fe04f8211f194a2c3b1eecd91b7"
tags: []

---

# Sensors & Bridges

## Overview

Two "world-facing" sensors feed the autopilot: a front-facing lidar
(modeled as a one-ray laserscan in Gazebo) and a forward camera. Rob's
code splits each into a **publisher** (subscribes Gazebo → publishes
ZMQ) and a **consumer** (subscribes ZMQ → exposes Python events). The
indirection lets multiple Python processes share one Gazebo subscription
and makes the autopilot transport-agnostic. Files:
`bt_app/sensors/{gz_lidar.py, gz_camera.py, sim_senors.py, harmonic_camera.py,
jetty_camera.py, gz_rtf_monitor.py}`.

## Key Decisions

- **Gazebo transport via `gz.transport13` + `gz.msgs10`.**
  Imports binary protobuf types directly (`from gz.msgs10 import
  laserscan_pb2`, `gz.msgs.image_pb2.Image`). Tightly bound to the
  Gazebo Harmonic SDK version.
- **Bounded queues, drop-on-full.** Both `GazeboLidarPublisher.scans` and
  `GazeboCameraSource` keep a `queue.Queue(maxsize=1)` and explicitly
  pop-then-push to keep only the latest sample (`gz_lidar.py:131-141`).
  Latest-wins is the right semantic for control loops.
- **ZMQ IPC, not TCP, for sensor topics.** Endpoints are
  `ipc:///tmp/bt_app.camera`, `ipc:///tmp/bt_app.ultrasonic_lidar`,
  `ipc:///tmp/bt_app.tracker_result` (`bt_app/common/__init__.py:7-11`).
  IPC because everything runs on the same host.
- **`SimSensors` is a thin consumer wrapper.** Subscribes to lidar topic,
  decodes the latest scan from a multipart `(topic, metadata, measurement)`
  message, emits `on_lidar_range(range_m, metadata)` (`sim_senors.py`).
  Same shape will be reused for camera frames.
- **Multipart message contract.** Publishers always send
  `[topic, json(metadata), json(measurement)]`. Consumers must drain
  with `recv_multipart` and keep only the most recent — same drop-old
  pattern as the publisher (`sim_senors.py:83-100`).
- **`gz_rtf_monitor.py` watches real-time factor** as a separate
  diagnostic — sim slowdown invalidates control timing.
- **Multiple camera variants.** `gz_camera.py` (canonical, Harmonic),
  `harmonic_camera.py`, `jetty_camera.py` — variants for different
  Gazebo distributions. Only one is active at a time.

## Constraints

- IPC socket paths are absolute (`/tmp/...`); two instances on the same
  host would collide. Multi-instance requires per-instance suffix.
- Publishers `bind`, consumers `connect`. If the consumer starts before
  the publisher, ZMQ buffers nothing — first messages may be missed.
- Bounded queue at 1 means a slow consumer drops frames silently — no
  metric / warning beyond a `logger.debug("Dropped lidar scan…")`.
- `_recv_latest_lidar` loops with `NOBLOCK` until empty — assumes the
  publisher will not deliver an infinite burst within one poll window.
- All metadata is JSON (`json.dumps(metadata).encode("utf-8")`); image
  pixels are raw bytes. No compression; raw RGB → tens of MB/s at HD.

## Open Questions

- Should we move to native gz pub/sub for camera frames to skip the
  JSON+raw-bytes detour? Latency vs. multi-consumer trade.
- No timestamp synchronization: each message carries metadata but
  there's no global clock alignment between camera and lidar. Visual
  controller doesn't fuse them, so it's fine today.
- Three camera files exist; clean up to one canonical on rebuild.
- `gz_rtf_monitor` is informational only — does the autopilot need to
  react when RTF drops (e.g., refuse arming)?
