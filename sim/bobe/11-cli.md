---
noteId: "df4219f04f8211f194a2c3b1eecd91b7"
tags: []

---

# CLI (`bti`)

## Overview

`bt_cli/` is a Typer-based CLI that lets a human (or a script) inspect
and tune the running autopilot from a terminal. Its sole transport is
the parameter ZMQ server at `tcp://127.0.0.1:5555` — there is no direct
control over flight, just live introspection of the parameter store.
Entry point `bti = bti_cli.main:app`. Subcommand `bti param` covers
`list / dump / save / describe / get / set`.

## Key Decisions

- **Typer for ergonomic CLI.** Auto-help, env-var bindings
  (`BTI_ZMQ_ENDPOINT`, `BTI_ZMQ_TIMEOUT_MS`), and short flags
  (`-e`, `-f`) come for free (`bti_cli/param/command.py:15-31`).
- **One transport, sync REQ/REP.** `ZmqRequestResponseTransport` opens a
  socket per call (`bt_cli/bti_cli/transport/zmq_transport.py`) — simple,
  no connection pooling.
- **Errors are typed.** `ZmqTransportTimeout` and `ZmqTransportError` are
  distinct so the CLI can produce a useful exit code (1) and stderr
  message instead of a traceback (`param/command.py:42-49`).
- **`echo_result` pretty-prints.** Dicts/lists → JSON-2-indent; scalars
  → bare repr (`command.py:52-56`). Pipeable.
- **Default endpoint matches `bt_app`'s server.**
  `tcp://127.0.0.1:5555`, 3 s timeout — both can be overridden per
  invocation.
- **Demo subcommand is wired but minimal.** `bti_cli/demo/command.py`
  exists alongside `param/`; structure leaves room for more namespaces
  (e.g., `bti state`, `bti rc`).

## Constraints

- The CLI assumes a parameter server is up. If `bt_app` isn't running,
  every command times out after `--timeout-ms`.
- No streaming / no subscribe — only one-shot REQ/REP. Watching a value
  needs `watch -n 0.5 bti param get foo`.
- Standalone install: `bti-cli` package is separate from `bt-app`. Useful
  if you want a thin shell environment, but installs duplicate the
  transport code.
- All values go over JSON; numbers lose precision at very large
  magnitudes (not an issue for control gains).

## Open Questions

- No "watch / subscribe" mode for live tuning feedback — would need a
  PUB channel from the parameter service.
- No batch / transaction support (`bti param set-many file.yaml`) — each
  set is one round-trip.
- Mock server in `bt_cli/tests/mock/` defines the response contract;
  worth promoting that to a shared schema.
- Should the CLI grow flight commands (`bti rc set …`)? Currently
  separation of concerns is "CLI tunes, behavior tree commands". If we
  want manual override during dev, the boundary needs to be redrawn.
