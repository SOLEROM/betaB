---
noteId: "e7af63904f8211f194a2c3b1eecd91b7"
tags: []

---

# GUI Scaffold

## Overview

`bt_gui/` is a **PyQt6 MVP scaffold** intended to grow into a live
telemetry / tuning GUI for the autopilot. As shipped, it's a placeholder:
three vertically stacked text boxes driven by a `RandomDataService` that
emits dummy values. No connection to the parameter service, MSP, or
sensors yet. Useful as a structural template for the eventual UI but not
on the critical path.

## Key Decisions

- **MVP (Model-View-Presenter) split.**
  `models/` (pure data), `services/` (background producers),
  `views/` (Qt widgets), `presenters/` (glue) — strict separation so
  Qt signals stay isolated from data models (`bt_gui/README.md`).
- **PyQt6.** Modern Qt, `QApplication` and `QMainWindow` based
  (`bt_gui/bt_gui/app.py:1-26`).
- **Service uses a background producer.** `RandomDataService` runs in a
  thread and pushes via Qt signal to the presenter. Pattern reusable for
  any data source (parameter pushes, MSP telemetry).
- **`aboutToQuit` cleanup.** App connects `qt_app.aboutToQuit` to
  `presenter.stop()` so threads shut down deterministically
  (`app.py:20`).
- **Test scaffolding present.** `tests/test_random_data_model.py`,
  `tests/mock_data.py` — establishes the pattern for unit-testing models
  without Qt event loop.

## Constraints

- Requires a display server. PyQt6 won't initialize on a headless
  container without X forwarding or `QT_QPA_PLATFORM=offscreen`.
- Hardcoded coupling to `RandomDataService` — no abstraction yet for
  swapping in real data sources.
- Run command (`venv/bin/python bt_gui/main.py`) presupposes the workspace
  venv layout from `bt_app`.

## Open Questions

- Will this become the production tuner, or stay a sketch? If it grows
  to consume the parameter service, it needs a `ZmqParameterClient`
  module shared with `bt_cli`.
- Should the GUI subscribe to a future telemetry PUB channel (FC state,
  altitude, lidar) — or pull via REQ/REP like the CLI? Push would map
  cleanly onto Qt signals.
- No design decisions on layout or feature set are committed in code —
  this is essentially a fresh slate to plan against.
