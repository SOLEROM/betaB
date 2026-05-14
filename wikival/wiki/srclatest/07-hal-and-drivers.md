---
type: architecture
status: stable
created: 2026-05-14
updated: 2026-05-14
tags: [hal, drivers, sensors, platform, abstraction]
source-commit: 6434dd725
---

# 07 — HAL & Drivers

Betaflight's hardware abstraction has **two layers**:

1. **`src/main/drivers/`** — abstract C headers. Function-pointer interfaces, opaque resource types. *Knows nothing about the MCU.*
2. **`src/platform/<arch>/`** — concrete implementations. Vendor SDK code, MCU register access. *Implements what the abstract layer declares.*

A third layer above them (`src/main/sensors/`) wraps drivers with calibration, alignment, and fusion logic. This page covers all three.

## The HAL pattern

The pattern is consistent across every driver family:

```
abstract header              concrete impl (one of)
─────────────────────         ─────────────────────────────────────────
drivers/serial.h              platform/STM32/serial_uart_stm32h7xx.c
drivers/serial_uart.h         platform/STM32/serial_uart_stm32f4xx.c
                              platform/AT32/serial_uart_at32f43x.c
                              platform/SIMULATOR/serial_tcp.c          (SITL)

drivers/timer.h               platform/STM32/timer_hal.c
                              platform/STM32/timer_ll.c
                              platform/AT32/timer.c
                              platform/PICO/timer.c

drivers/accgyro/accgyro_spi_icm426xx.h ─── used identically on every platform
                                            (only the SPI bus underneath differs)
```

Two mechanisms make this work:

**Function-pointer vtables.** The abstract type carries a pointer to a vtable of operations. Example from `drivers/serial.h`:

```c
struct serialPortVTable {
    void     (*serialWrite)(serialPort_t *p, uint8_t ch);
    uint32_t (*serialTotalRxWaiting)(const serialPort_t *p);
    uint8_t  (*serialRead)(serialPort_t *p);
    void     (*serialSetBaudRate)(serialPort_t *p, uint32_t baudRate);
    // …
};
typedef struct serialPort_s {
    const struct serialPortVTable *vTable;
    /* common state */
} serialPort_t;
```

`serialWrite(port, c)` is a macro/inline that dispatches through `port->vTable->serialWrite(port, c)`. Each transport (UART, USB VCP, soft-serial, TCP for SITL) provides its own vtable.

**Opaque resource types.** Hardware "things" (timers, DMA streams, SPI peripherals) are exposed as opaque `*_t` pointers to the abstract layer; only platform code knows what they really are.

```c
// drivers/dma.h
typedef struct dmaResource_s dmaResource_t;  // forward decl, no fields

// platform/STM32/dma_stm32h7xx.c
dmaResource_t *ref = (dmaResource_t *)DMA1_Stream0;            // cast in
DMA_Stream_TypeDef *stream = (DMA_Stream_TypeDef *)ref;        // cast out
```

This lets `drivers/dshot*` say "I want to use DMA stream X for this motor" without depending on whether `X` is an `LL_DMA_*` config struct or a `DMA_Stream_TypeDef *`.

## `drivers/` inventory

### Bus / fabric

| File / dir | Purpose |
|------------|---------|
| `bus.h`, `bus.c` | Top-level bus abstraction. `busDevice_t`, `extDevice_t`. `BUS_TYPE_SPI`, `BUS_TYPE_I2C`, `BUS_TYPE_QSPI`, `BUS_TYPE_OCTOSPI`, `BUS_TYPE_MPU_SLAVE`. |
| `bus_spi.h`, `.c`, `_impl.h`, `_types.h`, `_config.c` | SPI master driver. Segmented DMA transfers (`busSegment_t`), CS line management. |
| `bus_i2c.h`, `_impl.h`, `_types.h`, `_utils.c`, `_timing.c`, `_busdev.c`, `_soft.c` | I²C master driver. Both hardware and bit-banged variants. |
| `bus_quadspi.h`, `.c`, `_impl.h`, `_types.h` | QSPI for fast external flash. |
| `bus_octospi.h`, `.c`, `_impl.h` | OSPI (8-bit) flash. |

### Timer / DMA / interrupts

| File | Purpose |
|------|---------|
| `timer.h`, `timer_impl.h`, `timer_types.h` | Hardware-timer abstraction. `timerHardware_t` ties a GPIO `ioTag_t` to a timer + channel + DMA stream. Capture/compare callback registry. |
| `dma.h`, `dma.c`, `dma_impl.h`, `dma_reqmap.h` | DMA engine abstraction. Stream identifiers, channels, request-mux mapping. |
| `exti.h` | External interrupt controller (EXTI). Used for gyro DRDY pins. |
| `nvic.h` | Nested Vector Interrupt Controller — interrupt priority macros. |

### Motor / PWM output

| File | Purpose |
|------|---------|
| `motor.c`, `motor.h`, `motor_impl.h`, `motor_types.h` | Motor command distribution. Selects PWM / OneShot / MultiShot / DShot based on `motor_pwm_protocol`. |
| `pwm_output.h`, `_impl.h` | PWM generation contract. |
| `dshot.c`, `dshot.h`, `dshot_command.c`, `dshot_command.h` | DShot digital protocol (150/300/600/1200 kbps). Beacon, motor direction, settings via special commands. |
| `dshot_bitbang.h`, `dshot_bitbang_decode.c`, `_decode.h` | Software-bitbang DShot for inputs (bidirectional DShot telemetry decode). |
| `servo_impl.h` | Servo PWM driver. |

### Sensors

| Subdir | Devices |
|--------|---------|
| `drivers/accgyro/` | MPU6000, MPU6050, MPU6500, MPU9250, ICM20602, ICM20608G, ICM20649, ICM20689, ICM20602, ICM42605, ICM42688P, ICM45686, BMI160, BMI270, LSM6DSO, LSM6DSV16X, L3G4200D, L3GD20, BNO055, plus `accgyro_virtual.h` for SITL |
| `drivers/barometer/` | BMP085, BMP280, BMP388, BMP5xx, DPS310, LPS22DF, MS5611, QMP6988, 2SMPB-02B, virtual |
| `drivers/compass/` | AK8963, AK8975, HMC5883L, IST8310, LIS2MDL, LIS3MDL, MMC5603, MMC5983, QMC5883, virtual |
| `drivers/opticalflow/` | Optical flow sensors |
| `drivers/rangefinder/` | HC-SR04, TF-Mini, VL53L0X / L1X laser ToF, etc. |

Each device file pair `drivers/accgyro/accgyro_spi_icm42688.{c,h}` exposes a `bool icm42688SpiDetect(const extDevice_t *dev)` probe + `void icm42688SpiAccInit(accDev_t *acc)` / `…GyroInit(gyroDev_t *gyro)` initialisers. The `gyroDev_t` / `accDev_t` is filled with function pointers (`.readFn`) the sensor layer then polls.

### Storage

| File | Purpose |
|------|---------|
| `flash/` subdir + `flash.c`, `flash.h`, `flash_impl.h` | External SPI/QSPI flash chips. Drivers for M25P16, W25Q128, W25M, W25N (with paging), MX66UW1G45G, MT29F. |
| `sdcard.h`, `sdcard_impl.h`, `sdcard_standard.h` | SD card abstract layer. |
| `sdio.h`, `sdmmc_sdio.h` | SDIO peripheral driver (faster than SPI mode). |

### Display

| File | Purpose |
|------|---------|
| `max7456.c`, `max7456.h` | MAX7456 OSD chip (analog video overlay). SPI peripheral with character framebuffer. |
| `display.c`, `display.h`, `display_canvas.c`, `display_canvas.h` | Generic display abstraction. Character-grid (`display_t`) plus pixel canvas (`displayCanvas_t`). |
| `display_ug2864hsweg01.c` | Specific OLED driver. |
| `lcd_panel.h`, `lcd_panel/lcd_panel_font_5x7.h` | LCD panel abstraction. |
| `osd.h`, `osd_symbols.h` | OSD symbol codepoints (artificial horizon ticks, battery icon, etc.). |
| `fb_osd_impl.h` | Framebuffer-OSD impl interface (for HD/DJI OSD). |

### USB

| File | Purpose |
|------|---------|
| `usb_io.h` | USB peripheral GPIO setup. |
| `usb_msc.h` | USB Mass Storage Class (mounts SD/flash to host PC). |

### Misc

| File | Purpose |
|------|---------|
| `light_led.c/.h` | On-board status LED. |
| `light_ws2811strip.c/.h` | Addressable RGB strips (WS2811/SK6812/APA102). DMA-driven. |
| `sound_beeper.h` | Beeper / buzzer GPIO. |
| `transponder_ir.h`, `_ilap.h`, `_arcitimer.h`, `_erlt.h` | IR transponders for racing time-keeping. |
| `vtx_common.h`, `vtx_rtc6705.h`, `_soft_spi.h` | VTX register-level drivers. (Higher-level VTX protocols in `io/`.) |
| `camera_control.c/.h/_impl.h` | Camera trigger / gimbal trigger. |
| `pinio.h` | Programmable I/O channels — generic configurable GPIO outputs. |
| `pin_pull_up_down.h` | Per-pin pull-up/down config. |
| `adc.c`, `adc.h` | ADC abstraction (battery voltage, current, RSSI, RX-grade analog inputs). |
| `inverter.c`, `inverter.h` | Signal inverter for half-duplex UART (used by SmartPort, F.Port). |
| `mco.h` | Master Clock Output (drives external gyro clock from MCU). |
| `gyro_clkin.h` | Gyro external-clock input config. |
| `memprot.h` | MPU (memory protection unit) region setup. |
| `persistent.h` | Backup-domain registers (survive reset). Used for bootloader handoff. |
| `system.h` | `systemReset()`, `systemResetToBootloader()`. |
| `time.h` | `micros()`, `millis()`, `delay()`. Platform-implemented in `system_<MCU>.c`. |
| `io.c/.h/_def.h/_impl.h/_types.h`, `io_preinit.c` | GPIO pin abstraction. `ioTag_t` (8-bit encoded GPIO port+pin) is the universal pin identifier. |
| `resource.h` | **Resource ownership registry.** `OWNER_MOTOR`, `OWNER_SERVO`, `OWNER_SERIAL_TX`, `OWNER_ADC_BATT`, … 50+ owners. Prevents double-allocating a pin. |
| `buttons.c/.h` | GPIO buttons (rare; used on boards with physical buttons). |
| `stack_check.c` | Stack overflow detection (fills stack with sentinel, checks against running pointer). |
| `audio.h` | Audio DAC abstraction (used by `io/pidaudio.c`). |
| `buf_writer.c/.h` | Buffered byte-writer (for logging, MSP responses). |

### CAN

| File | Purpose |
|------|---------|
| `can/can.h`, `can_impl.h`, `can_types.h` | CAN bus abstraction. Used by DroneCAN node implementation in `io/dronecan/`. |

### RX (driver-level)

| File | Purpose |
|------|---------|
| `drivers/rx/` | Receiver radios with direct SPI/wire interface — the RF chip drivers themselves. CC2500, CYRF6936, A7105, NRF24L01, SX127x/SX1280 (ExpressLRS). Higher-level protocol parsing lives in `src/main/rx/`. |

## How abstract → concrete actually links

Example: `serialWrite(uartPort, byte)` from `io/serial.c`.

1. **Compile-time.** `src/main/io/serial.c` calls `serialWrite()` → it expands through `serial.h` into `port->vTable->serialWrite(port, byte)`.
2. **Build sees no concrete impl in `src/main/`.** The `.o` from `serial.c` has an unresolved indirect call to a function pointer.
3. **Platform code provides the vtable.** `src/platform/STM32/serial_uart_stm32h7xx.c` defines a static `uartVTable` and points `uartPort_t::vTable` to it during `uartOpen()`.
4. **Linker pulls the right platform code** because `MCU_COMMON_SRC` in the MCU's `mk/<MCU>.mk` lists the platform `.c` files.

So the abstraction is dynamic-dispatch at runtime, but it's static-dispatch at link time: only one impl per peripheral type is linked into the final binary (the one for this MCU family).

## `src/platform/<arch>/` layout (STM32 example)

```
src/platform/STM32/
├── mk/
│   ├── STM32_COMMON.mk      ← shared by all STM32 subfamilies
│   ├── STM32F4.mk           ← F4-specific
│   ├── STM32F7.mk
│   ├── STM32G4.mk
│   ├── STM32H7.mk
│   └── ...
├── link/
│   ├── stm32_flash_f405.ld  ← linker scripts per MCU
│   ├── stm32_flash_f7x2_1024k.ld
│   ├── stm32_flash_h743.ld
│   └── ...
├── startup/                 ← vendor startup_*.s files
├── include/                 ← STM32-only headers (CMSIS shims, platform.h)
├── target/
│   ├── STM32F405/{target.h, target.mk}
│   ├── STM32H743/{target.h, target.mk}
│   └── ...                  ← one dir per MCU variant
├── adc_stm32f4xx.c, adc_stm32h7xx.c, …    ← per-subfamily peripheral drivers
├── bus_i2c_*.c, bus_spi_*.c, ...
├── dma_stm32h7xx.c, dma_reqmap_mcu.c
├── timer_hal.c, timer_ll.c                ← HAL or LL flavour selected by MCU
├── serial_uart_stm32f4xx.c, _stm32h7xx.c
├── serial_usb_vcp.c
├── vcp_hal/                               ← USB CDC stack
├── pwm_output_dshot_hal.c, _ll.c
├── persistent_hal.c, persistent_stdperiph.c
└── system_stm32f4xx.c, ...
```

**Three driver "flavours"** show up in STM32:

- **HAL** — high-level ST HAL (`HAL_UART_Transmit_DMA`, etc.). More portable, slower.
- **LL** — Low-Level (`LL_USART_TransmitData8`). Closer to registers, faster, smaller.
- **stdperiph** — legacy "Standard Peripheral Library" (pre-HAL). Maintained only for older subfamilies.

The choice is made in the MCU's `mk/<MCU>.mk`. F4/F7 mostly use HAL or LL; H7 prefers HAL2 (newer). Newer MCUs (H5, N6, C5) are HAL-only.

## Sensor selection at runtime — gyro example

`src/main/sensors/gyro_init.c` walks the configured `gyroDeviceConfig[i].busType` + bus index + chip-select pin and probes each known driver in order:

```
for each enabled sensor slot:
   if config says SPI MPU6000:
     gyroSpiMpu6000Detect(extDevice) → read WHO_AM_I over SPI
     if matches 0x68 -> populate gyroDev_t with mpu6000 fns
   else if config says SPI ICM42688P:
     icm42688pSpiDetect(extDevice) → read register 0x75
     if matches 0x47 -> populate gyroDev_t with icm42688p fns
   ...
```

The `gyroHardware_e` enum in `drivers/accgyro/accgyro.h` lists every supported part. On detection, `gyroDev_t` is populated:

```c
typedef struct gyroDev_s {
    sensorGyroInitFuncPtr   initFn;       // sensor-specific init
    sensorGyroReadFuncPtr   readFn;       // sensor-specific read
    float scale;                          // ADC → deg/s factor (depends on full-scale range)
    extDevice_t dev;                      // bus + CS
    busSegment_t segments[N];             // pre-built DMA descriptors
    int16_t gyroADCRaw[XYZ];              // last raw values
    /* ... */
} gyroDev_t;
```

After init, the gyro task only calls `readFn(gyroDev)` — it doesn't know or care which chip it's talking to.

`gyro_t` (in `sensors/gyro.h`) wraps up to 2 `gyroDev_t` for dual-gyro setups (which average or select). It also owns the filter chain — LPF1, LPF2, optional static notches, RPM-notches, dyn-notch FFT, board alignment rotation. The output is `gyroADCf[XYZ]` in deg/s, body frame, post-filter — what `pidController()` reads.

## `sensors/` — the fusion layer

| File | Role |
|------|------|
| `gyro.c`, `gyro_init.c`, `gyro_filter_impl.c`, `gyro.h` | Multi-sensor gyro fusion + filter chain. ~2500 LOC. |
| `acceleration.c`, `acceleration_init.c` | Accel fusion + calibration. `accADC[XYZ]` in g. |
| `barometer.c`, `.h` | Baro sampling, altitude estimation, vario. |
| `compass.c`, `.h` | Compass sampling, hard/soft-iron calibration, heading. |
| `current.c`, `current_ids.h` | Current sensor abstraction. ADC and ESC-telemetry sources. |
| `voltage.c`, `voltage_ids.h` | Voltage meter abstraction. ADC and ESC-telemetry sources. Multi-cell tracking. |
| `battery.c`, `.h` | Battery state machine. Charge tracking, alarm thresholds, beeper triggers. |
| `adcinternal.c`, `.h` | MCU-internal ADC channels (temperature, VREFINT). |
| `esc_sensor.c`, `.h` | ESC telemetry decode (KISS, BLHeli32, bidirectional DShot). |
| `opticalflow.c`, `.h` | Optical flow sensor wrapping. |
| `rangefinder.c`, `.h` | Distance-sensor wrapping. |
| `boardalignment.c`, `.h` | Roll/pitch/yaw rotation of all body-frame sensor data. |
| `initialisation.c`, `.h` | Top-level `sensorsInit()` orchestrating gyro/acc/baro/mag/rangefinder probe. |
| `sensors.c`, `.h` | `enabledSensors` bitmask, `sensors()` query macro. |

## Target wiring — pins to peripherals

A target's `target.h` declares the **capabilities** of the MCU + board:

```c
#define USE_SPI_DEVICE_1
#define USE_SPI_DEVICE_2
#define USE_UART1
#define USE_UART2
#define USE_UART3
#define USE_I2C_DEVICE_1
#define USE_ADC
#define USE_TIMER
#define FLASH_PAGE_SIZE 0x20000
```

Some legacy / non-unified targets also hard-code pin assignments here. For unified targets (most of master), pins come from the `CONFIG` layer at runtime — they're stored in PG records like `serialPinConfig_t` and `spiPinConfig_t`:

```c
serialPinConfig_t {
    ioTag_t ioTagTx[SERIAL_PORT_COUNT];
    ioTag_t ioTagRx[SERIAL_PORT_COUNT];
    ioTag_t ioTagInverter[SERIAL_PORT_COUNT];
};
```

The board's `config.h` sets defaults via PG reset templates: `PA9` for `UART1_TX`, `PA10` for `UART1_RX`, etc. User can override at the CLI.

`fullTimerHardware[]` (per target) is the master list of every timer/channel/pin combination available on the MCU. The motor driver consults it to pick channels for motor outputs based on configured motor pin order.

## Timer/DMA model for DShot (concrete walkthrough)

DShot encodes each motor command as 16 bits transmitted MSB-first at one of four bitrates. Implementation on STM32:

1. **Timer setup.** Each motor pin is on a hardware timer's CC channel, configured in PWM mode 1.
2. **Bit stream precompute.** For each motor, build a 16-element `uint16_t[]` of CCR values: high pulse width = "1" or "0".
3. **DMA fire.** Start a DMA stream from the precomputed buffer → `TIM->CCR[ch]` register. Each timer update event consumes one buffer entry, generating the pulse.
4. **Completion ISR.** When the DMA transfer-complete IRQ fires, reset the GPIO and (for bidirectional DShot) start an EXTI capture for the ESC's response telemetry.

Files: `drivers/pwm_output.h` (abstract) → `platform/STM32/pwm_output_dshot_hal.c` (concrete HAL) or `_ll.c` (concrete LL). The DMA request mapping (`TIM2_CH1` → `DMA1_Stream5` + `DMAMUX_REQ_TIM2_CH1`) lives in `dma_reqmap_mcu.h` per MCU subfamily.

## Resource ownership

`drivers/resource.h` declares ~50 `OWNER_*` IDs. Any GPIO pin claim goes through `resourceClaimPin(ioTag, OWNER_MOTOR1)` which records the owner in a global table; subsequent claims by a different owner trigger a failure. CLI `resource` command exposes the table.

Why this matters: pins on a typical 40-pin MCU are shared between many possible peripherals (a single pin could be UART2_TX, TIM1_CH4, or ADC1_CH7). Without resource tracking, two features would silently conflict and one would not work. With it, the firmware refuses to bring up a feature whose pin is already taken and reports the conflict in the boot log.

## Conditional compilation summary

Build-time gates dominate this layer. A few common flags:

| Flag | Effect |
|------|--------|
| `USE_GYRO_SPI_ICM42688P` | Include the ICM42688P driver in the build |
| `USE_BARO`, `USE_MAG`, `USE_RANGEFINDER` | Sensor families |
| `USE_DSHOT`, `USE_DSHOT_BITBANG`, `USE_DSHOT_TELEMETRY` | Motor protocols |
| `USE_OSD`, `USE_OSD_HD`, `USE_MAX7456` | OSD transports |
| `USE_USB_MSC`, `USE_SDCARD`, `USE_FLASHFS` | Storage backends |
| `USE_VTX_TRAMP`, `USE_VTX_SMARTAUDIO`, `USE_VTX_MSP` | VTX protocols |
| `USE_LEDSTRIP`, `USE_BEEPER`, `USE_TRANSPONDER` | Misc outputs |

Boards turn these on/off in their `config.h`. RAM/flash-constrained F4 boards typically disable advanced filters and some sensors; H7 boards have everything enabled.

## Adding a new sensor (quick recipe)

1. Add `accgyro_spi_<name>.c/.h` under `drivers/accgyro/`.
2. Implement `<name>SpiDetect()`, `<name>SpiAccInit()`, `<name>SpiGyroInit()`, read/init function pointers.
3. Add an enum value to `gyroHardware_e` in `accgyro.h`.
4. Add probe call to `gyroDetect()` in `sensors/gyro_init.c`.
5. Add `USE_GYRO_SPI_<NAME>` to relevant board `config.h` (or generic target).
6. Add the source file to `mk/source.mk` or rely on its `wildcard accgyro/*.c` glob.

For a new MCU family, repeat the same template under `src/platform/<NEW_ARCH>/` and add the `mk/<NEW_ARCH>.mk` fragment. See [[13-modification-guide]].

## Where things live

| Concern | File |
|---------|------|
| Driver abstraction roots | `src/main/drivers/*.h` |
| Sensor fusion | `src/main/sensors/*` |
| Platform implementations | `src/platform/<arch>/*.c` |
| HAL flavour choice | `src/platform/<arch>/mk/<MCU>.mk` |
| Linker script | `src/platform/<arch>/link/*.ld` |
| Target capabilities | `src/platform/<arch>/target/<MCU>/target.h` |
| Pin assignments | board `config.h` (under `src/config/configs/<BOARD>/`) |
| Resource registry | `src/main/drivers/resource.h` |
| Pin abstraction | `src/main/drivers/io.h` (`ioTag_t`) |
