---
type: architecture
status: stable
created: 2026-05-14
updated: 2026-05-14
tags: [scheduler, boot, init, tasks, realtime]
source-commit: 6434dd725
---

# 04 — Boot Sequence & Scheduler

The two functions you need to understand the firmware are `main()` and `scheduler()`. Everything else is called from one of them.

## Boot sequence (3 phases)

`src/main/main.c` is short — under 150 lines. The control flow:

```
main(argc, argv)
  └ systemInit()                  ← clocks, NVIC, multicore bootstrap
  └ printfSerialInit()            ← optional debug serial
  └ initPhase1()                  ← drivers/* low-level: GPIO, buses, ext-flash
  └ initPhase2()                  ← sensor probe, config load, peripheral setup
  └ initPhase3()                  ← high-level: IMU/mixer/PID/scheduler tasks armed
  └ run()
       └ while (true) scheduler();
```

`initPhase{1,2,3}()` live in **`src/main/fc/init.c`**. The split exists because some MCUs run init phases on different cores (Pico, multi-core ESP32) via `multicoreExecuteBlocking()`.

### What runs in each phase

**Phase 1 — `initPhase1()`** (must complete before sensors are touched):

- `ioInit()` — pin/GPIO subsystem
- `EXTIInit()` — external interrupt controller
- Bus init: `spiPreinit()`, `spiInit(spiConfig(i))` for each enabled SPI device; same for I2C.
- External flash init (for blackbox / OTP / config storage)
- LED, beeper, button GPIO
- `cliBootLogEntryAdd()` calls trickle in throughout to populate the boot log

**Phase 2 — `initPhase2()`** (sensors + config):

- `readEEPROM()` — load all Parameter Groups from flash → see [[12-config-and-pg]]
- `validateAndFixConfig()` — clamp out-of-range settings, resolve feature conflicts
- `sensorsInit()` — probe gyro/acc/baro/mag, populate `gyroDev_t` etc.
- USB, OSD chip (MAX7456 over SPI), SD card init
- Set `enabledSensors` bitmask via `sensorsSet(SENSOR_GYRO|SENSOR_ACC|…)`

**Phase 3 — `initPhase3()`** (flight-ready):

- `imuInit()` — DCM/quaternion state zeroed
- `mixerInit()` — motor mixer table loaded from config
- `pidInit()` — PID profile activated, filter coefficients computed
- `failsafeInit()`, `rxInit()`, `telemetryInit()`, `blackboxInit()`
- `scheduleInit()` — populate the task queue
- `tasksInit()` — call `setTaskEnabled()` per task to mark which run

Then `run()` enters the scheduler loop. From this point on the firmware is purely event/timer driven.

## The scheduler

Single file pair: **`src/main/scheduler/scheduler.c` + `scheduler.h`**. About 700 lines total. It is a hand-rolled cooperative scheduler — no FreeRTOS, no preemption, no threads. The only "preemption" comes from hardware ISRs (gyro DRDY, DMA-complete, USART RX).

### Task priority model

From `scheduler.h:62-70`:

```c
typedef enum {
    TASK_PRIORITY_REALTIME    = -1,  // Gyro/Filter/PID — bypass dynamic priority
    TASK_PRIORITY_LOWEST      =  1,
    TASK_PRIORITY_LOW         =  2,
    TASK_PRIORITY_MEDIUM      =  3,
    TASK_PRIORITY_MEDIUM_HIGH =  4,
    TASK_PRIORITY_HIGH        =  5,
    TASK_PRIORITY_MAX         =  255
} taskPriority_e;
```

**`TASK_PRIORITY_REALTIME` is special** — these tasks (`TASK_GYRO`, `TASK_FILTER`, `TASK_PID`) execute on every scheduler iteration, gated by counter modulo arithmetic, not by the dynamic-priority queue. The whole rest of the system runs in the time between gyro ticks.

### Task structure

From `scheduler.h:209-244`:

```c
typedef struct {
    const char *taskName;
    const char *subTaskName;
    bool  (*checkFunc)(timeUs_t currentTimeUs, timeDelta_t currentDeltaTimeUs);
    void  (*taskFunc)(timeUs_t currentTimeUs);
    timeDelta_t desiredPeriodUs;       // e.g. TASK_PERIOD_HZ(100) = 10 000 µs
    const int8_t staticPriority;
} task_attribute_t;

typedef struct {
    task_attribute_t *attribute;
    uint16_t  dynamicPriority;         // ages up between executions
    timeDelta_t taskLatestDeltaTimeUs;
    timeUs_t  lastExecutedAtUs;
    timeUs_t  lastDesiredAt;
    float     movingAverageCycleTimeUs;
    timeUs_t  anticipatedExecutionTime;
} task_t;
```

The **two-function model** is the heart of the scheduler:

- `checkFunc` — runs every scheduler tick, returns `true` if the task wants to run. May also signal anticipated work via the `currentDeltaTimeUs` argument.
- `taskFunc` — the actual work. Runs only when selected.

Tasks like `TASK_RX` use `checkFunc` to test "did a new RC frame arrive?" so they don't waste CPU when nothing has happened.

### The task table

`src/main/fc/tasks.c` declares the `tasks[]` array. Each entry uses `DEFINE_TASK(name, subName, checkFn, taskFn, periodUs, priority)`:

```c
[TASK_GYRO]         = DEFINE_TASK("GYRO",      NULL, NULL, taskGyroSample,
                                  TASK_GYROPID_DESIRED_PERIOD, TASK_PRIORITY_REALTIME),
[TASK_FILTER]       = DEFINE_TASK("FILTER",    NULL, NULL, taskFiltering,
                                  TASK_GYROPID_DESIRED_PERIOD, TASK_PRIORITY_REALTIME),
[TASK_PID]          = DEFINE_TASK("PID",       NULL, NULL, taskMainPidLoop,
                                  TASK_GYROPID_DESIRED_PERIOD, TASK_PRIORITY_REALTIME),
[TASK_ATTITUDE]     = DEFINE_TASK("ATTITUDE",  NULL, NULL, imuUpdateAttitude,
                                  TASK_PERIOD_HZ(100),         TASK_PRIORITY_MEDIUM),
[TASK_RX]           = DEFINE_TASK("RX",        NULL, rxUpdateCheck, taskUpdateRxMain,
                                  TASK_PERIOD_HZ(33),          TASK_PRIORITY_HIGH),
[TASK_SERIAL]       = ...
[TASK_BATTERY_VOLTAGE]= ...
[TASK_BARO]         = ...
[TASK_GPS]          = ...
[TASK_TELEMETRY]    = ...
[TASK_OSD]          = ...
[TASK_LEDSTRIP]     = ...
[TASK_BLACKBOX]     = ...
[TASK_CMS]          = ...
// ~40 tasks total
```

A task is **enabled** (gets scheduled) only if `setTaskEnabled(TASK_X, true)` was called during init. This is how feature flags propagate to scheduling — if `USE_GPS` is off at compile time, `setTaskEnabled(TASK_GPS, true)` is never called and the task slot is dead.

### The scheduler main loop (`scheduler()`)

`src/main/scheduler/scheduler.c:486-599`. Pseudocode:

```c
void scheduler(void) {
    if (gyroEnabled) {
        // 1. Compute next gyro deadline using the CPU cycle counter
        nextTargetCycles = lastTargetCycles + desiredPeriodCycles;

        // 2. Busy-wait until the deadline (sub-microsecond precision)
        while (cmpTimeCycles(nextTargetCycles, getCycleCounter()) > 0) {}

        // 3. Execute realtime chain in order, gated by counters
        schedulerExecuteTask(TASK_GYRO, currentTimeUs);
        if (gyroFilterReady())
            schedulerExecuteTask(TASK_FILTER, currentTimeUs);
        if (pidLoopReady())
            schedulerExecuteTask(TASK_PID, currentTimeUs);
    }

    // 4. Pick best non-realtime task to run in remaining slack
    task_t *bestTask = NULL;
    for each waiting task t:
        if (t.attribute->checkFunc && !t.attribute->checkFunc(...))
            continue;            // task doesn't want to run
        t.dynamicPriority += t.attribute->staticPriority;
        if (t.dynamicPriority > best) { bestTask = t; }

    // 5. Only run it if we have time before the next gyro deadline
    if (bestTask && anticipatedExecutionTime(bestTask) < remainingCycles) {
        schedulerExecuteTask(bestTask, currentTimeUs);
        bestTask.dynamicPriority = 0;
    }
}
```

**Key invariants:**

- The gyro deadline is never missed unless the CPU is genuinely overloaded; if it is, "late task" counters increment.
- `dynamicPriority` aging ensures no task starves — a `TASK_PRIORITY_LOW` task that has waited long enough will eventually outscore a `TASK_PRIORITY_HIGH` task that just ran.
- `anticipatedExecutionTime` is a moving average of past runs; the scheduler skips a task if it estimates the task won't finish before the next gyro tick.

### Gyro-driven rates

The flight loop has two configurable rates:

- **Gyro sample rate** — set by sensor + `pid_process_denom = N`. Common: 8 kHz raw gyro, denom=1.
- **PID loop rate** — gyro rate / `activePidLoopDenom`. Common: 4 kHz (denom=2) or 2 kHz (denom=4).

These live in `sensors/gyro.h:125` (`activePidLoopDenom`) and gate two helper functions:

```c
bool gyroFilterReady(void) {
    return (pidUpdateCounter % activePidLoopDenom) == 0;
}
bool pidLoopReady(void) {
    return (pidUpdateCounter % activePidLoopDenom) == (activePidLoopDenom / 2);
}
```

So with denom=2 and gyro at 8 kHz: filter runs at 4 kHz aligned to gyro samples 0,2,4,…; PID runs at 4 kHz offset by one gyro sample (1,3,5,…). The half-period offset gives filtering time to settle before the PID consumes its output.

`TASK_GYROPID_DESIRED_PERIOD` is set per-platform — STM32F7/H7 default to 125 µs (8 kHz), lower-end MCUs to 1000 µs (1 kHz). See `src/platform/STM32/include/platform/platform.h`.

### Task execution timing (per cycle, 8 kHz)

```
t=0    µs   gyro DRDY interrupt → bus DMA fetches 6×i16 over SPI
t=5    µs   schedulerExecuteTask(TASK_GYRO)    raw → gyro state
t=15   µs   schedulerExecuteTask(TASK_FILTER)  LPF+notch+dyn-notch
t=45   µs   schedulerExecuteTask(TASK_PID)     subTaskRcCommand
                                               + subTaskPidController
                                               + subTaskMotorUpdate
                                               + subTaskPidSubprocesses
t=110  µs   bestTask = pickNonRealtime()
t=120  µs   schedulerExecuteTask(bestTask)     (RX, OSD, telemetry, …)
t=125  µs   next gyro DRDY
```

So the gyro+filter+PID chain consumes ~85 µs (68 % CPU at 8 kHz). The remaining ~40 µs/cycle is used for everything else, with the scheduler picking one task per cycle.

### Performance instrumentation

The scheduler keeps moving averages per task: cycle time, execution time, deadline misses. `cliCmd("tasks")` dumps the live table. The CMS "STATS" submenu also exposes CPU load. Important fields:

- `task.taskLatestDeltaTimeUs` — gap between this run and last
- `task.movingAverageCycleTimeUs` — 8-sample EMA
- `task.anticipatedExecutionTime` — estimated next-run cost (for slack check)
- Global `taskTotalExecutionTime`, `cpuPercentageInUse`

### Anti-starvation: aging and dynamic priority

Each iteration the scheduler adds `staticPriority` to `dynamicPriority` for every waiting task. The task with the highest `dynamicPriority` that fits in the slack window runs, then has its `dynamicPriority` reset to 0. This gives high-priority tasks frequent service while still letting low-priority tasks make progress.

`schedulerConfig()->rxRelaxDeterminism` is an "expedite" knob — when an RX check function fails repeatedly (no frame arrived), the scheduler scales down its estimated cost by 0.9× to make it easier to schedule next time. Lets RX recover quickly when frames start flowing again.

## What the scheduler does NOT do

- **No preemption.** A task that runs long blocks everything else until it returns. Tasks must never spin-wait for I/O.
- **No memory allocation.** All task state is static.
- **No priority inversion.** No locks, no shared mutexes — tasks pass data via plain globals and the cooperative model guarantees no two tasks run concurrently.
- **No coroutines.** Tasks have no stack of their own; they share the main thread's stack.
- **No real-time guarantee for non-realtime tasks.** A heavy CMS render can briefly push OSD below its desired 60 Hz; the only hard guarantee is on the gyro chain.

## Where things live

| Concern | File |
|---------|------|
| Boot orchestration | `src/main/main.c` |
| Phase 1/2/3 init | `src/main/fc/init.c` |
| Task table | `src/main/fc/tasks.c` |
| Scheduler internals | `src/main/scheduler/scheduler.c`, `.h` |
| Gyro task body | `src/main/fc/core.c:taskGyroSample` |
| Filter task body | `src/main/fc/core.c:taskFiltering` |
| PID task body | `src/main/fc/core.c:taskMainPidLoop` |
| Sensor selection | `src/main/sensors/gyro_init.c` |
| Rate constants | `src/platform/STM32/include/platform/platform.h:TASK_GYROPID_DESIRED_PERIOD` |

Next: [[05-flight-core-loop]] expands what each subtask in `taskMainPidLoop` actually does.
