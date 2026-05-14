---
noteId: "d44a54e04f8211f194a2c3b1eecd91b7"
tags: []

---

# Parameter Service

## Overview

`bt_app/parameters/` provides a **typed, limit-checked, hot-reloadable**
parameter store backed by YAML and accessible over ZMQ REQ/REP. It is
the canonical place for any value a controller might want to tune live —
PID gains, target altitudes, sign flips, RC ranges. Layered as
`ParameterStorage` (YAML load/save + dict-of-models) →
`ParameterService` (validation + change events) → `ZmqParameterServer`
(JSON-over-ZMQ-REP). The top-level `Parameters` facade owns a background
server thread.

## Key Decisions

- **YAML is the source of truth.** Each parameter declares
  `type` (int/float/str/bool/array) and `limits` (min/max or `options`)
  (`config/parameter_storage_example.yaml`). Defaults come from YAML;
  `save()` writes back to the file.
- **Hot reload through pub-sub-like events.**
  `ParameterService.on_parameter_changed` (`service.py:42`) is an event
  with `(name, value)`. Every controller subscribes and updates the right
  attribute on change (`takeoff_controller.py:55-64`,
  `hover_yaw_controller.py:136-150`, `visual_controller.py:446-451`).
- **Set fires the event before storage.** The service emits *first*,
  then stores (`service.py:41-43`) — this matters if storage rejects an
  out-of-range value; in practice the storage layer validates and may
  raise, but the event has already fired. Subtle ordering bug or
  intentional?
- **In-process declare/get/set.** `Parameters.declare(name, default,
  limits, value_type)` lets behaviors lazily register parameters they
  need (e.g., `VisualTrack.__init__` declares
  `visual.tracking_timeout_s`, `flight_tree.py:185-187`).
- **ZMQ wire protocol is a tiny JSON RPC.**
  Request: `{namespace:"param", action:"list|dump|save|describe|get|set",
  params:{...}}` → Response: `{ok: true, result: ...}` or
  `{ok: false, error: ...}` (`zmq_server.py:54-69`).
- **REP socket; one request at a time, 100 ms poll.** Single endpoint
  `tcp://127.0.0.1:5555` (`main.py:30-32`).
- **CLI parses string values opportunistically.** `_parse_value` tries
  `json.loads` first, falls back to string (`zmq_server.py:105-112`) —
  so `param set x 3.5` works.

## Constraints

- Single global parameter namespace per server (`"param"`); the protocol
  reserves `namespace` for future namespaces but nothing else exists.
- `Parameters` server runs on a daemon thread inside `bt_app`. If the
  main thread exits, the server dies with it.
- No authentication. Anyone with TCP access can flip any parameter
  (including arm flags via gain manipulation).
- Limits are checked on `set`, but type coercion is loose — setting a
  float into an int parameter is unspecified.
- Tests in `bt_cli/tests/mock/mock_param_server.py` and
  `parameter_storage_example.yaml` define the schema in practice; there
  is no JSON Schema doc.

## Open Questions

- Persistence: `save()` overwrites the YAML on disk — when, and by whom?
  No autosave; the user must call `param save` explicitly. Risk of
  losing tuned values on container restart.
- Concurrent writes (CLI + controller subscriber both updating the same
  param) — last write wins, but order is non-deterministic.
- No history / audit trail of parameter changes.
- Two YAMLs exist: `parameters.yaml` (generic camera/controller) vs.
  `parameter_storage_example.yaml` (actual flight parameters). Which is
  canonical? `main.py` defaults to the latter.
- Should arming-related state ever be a parameter? Currently no — but
  the boundary is undefined.
