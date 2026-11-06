# Deep Sleep Deprived: firmware and app architecture

**Board:** E2 Axle Strain Gauge, RevB (EFR32BG24B210F1024IM48)
**Status:** draft for team review. Comment on the pull request. Review due Sun 2026-11-08.
**Scope:** the custom board's firmware first. The dev-kit build for the Dec 1 in-class demo is a later task (see [§10](#10-tasks)).

Items marked **VERIFY** are assumptions. Check them against a datasheet or on the board before coding against them.

---

## 1. What the first version must do

From the 2026-11-06 team meeting:

1. Read the strain-gauge Wheatstone bridge through the ADS122C04 ADC.
2. Store the samples in the W25Q128 SPI flash.
3. Send stored data over Bluetooth LE to a laptop. The laptop runs a localhost web page that uses Web Bluetooth in Chrome. No native app.
4. Report energy-harvesting status: supercap / V_SYS voltage and LTC3588 PGOOD.
5. Leave power-consumption measurement off the board. It is an external test: shunt resistor or the Pro Kit AEM.

Fixed first-pass parameters, all compile-time constants in `app_config.h`:

| Parameter | Value |
|---|---|
| Strain sample rate | 10 Hz |
| Sample / stream cycle | 60 s of sampling, then a streaming step |
| Bridge | Quarter bridge, 120 Ω gauge, gauge factor 2.0 (**VERIFY** the GF on the ITLL gauge packet) |
| Data kept on the board | Raw ADC codes. The laptop converts to strain. |

Grading note: the professor gives credit for showing what didn't work and how it was debugged. So the firmware logs every failure and keeps counters, rather than failing silently.

---

## 2. Hardware the firmware sees

Taken from the PcbDoc at commit `2093131` (2026-11-03). Pins moved on 2026-10-29, so re-check this table if the board changes again.

### 2.1 MCU pin map

| BG24 pin | Port | Net | Use |
|---|---|---|---|
| 1 | PC00 | PMIC_PGOOD | LTC3588 power good, input |
| 2 | PC01 | I2C_SCL | ADS122C04 (4.7 k pull-up to V_MEASURE) |
| 3 | PC02 | I2C_SDA | ADS122C04 (4.7 k pull-up to V_MEASURE) |
| 4 | PC03 | ADC_DRDY_N | ADS122C04 data ready, active low |
| 5 | PC04 | MEASURE_EN | U3 TPS22919 → V_MEASURE (ADC + bridge) |
| 6 | PC05 | FLASH_EN | U4 TPS22919 → V_FLASH (W25Q128) |
| 7 | PC06 | FLASH_CS_N | Flash chip select (10 k pull-up to V_FLASH) |
| 8 | PC07 | SPI_MISO | Flash DO |
| 9 | PC08 | SPI_MOSI | Flash DI |
| 10 | PC09 | SPI_SCK | Flash CLK |
| 23 | PB00 | CAP_SENSE | V_SYS × 330 k / (1 M + 330 k), to the IADC |
| 27–29 | PA01–PA03 | SWCLK / SWDIO / SWO | Debug header J5 |
| 11–12 | — | HFXTAL | 39 MHz crystal (ECS-390-CDX-1932) |
| 37 | — | VREGSW | DCDC enabled (2.2 µH L2). DCDC feeds DVDD, and RFVDD/PAVDD through ferrites. |

All other GPIOs are unconnected. There is **no LED, button, UART or 32 kHz crystal**.

### 2.2 Power tree

```
Piezo/AC gen (J1) ─► LTC3588-1 (3.3 V out) ─► V_HARVEST ─┐
                                                          ├─ J3 jumper ─► V_SYS ─┬─ BG24 (AVDD, IOVDD, VREGVDD)
Bench 3.3 V (J4) ───────────────────────────► V_DEV_3V3 ─┘                      ├─ J2 jumper ─► 1 F / 5.4 V supercap (V_STORE)
                                                                                 ├─ U3 switch ─► V_MEASURE (ADC + bridge)
                                                                                 ├─ U4 switch ─► V_FLASH (W25Q128)
                                                                                 └─ R1/R2 divider ─► CAP_SENSE (PB00)
```

- The supercap sits on V_SYS when J2 is fitted. While the harvester keeps up, V_SYS is held at 3.3 V. When the load exceeds the harvest, V_SYS droops with the supercap, and the BG24 runs down to 1.71 V.
- **That droop is the energy signal.** CAP_SENSE plus PGOOD replace the "power in vs. power out" data the LTC3588 can't report.
- The CAP_SENSE divider draws about 2.5 µA all the time (3.3 V / 1.33 MΩ). Include it in the sleep-current budget.

### 2.3 Hardware facts that drive the design

1. **PGOOD and ADC_DRDY_N are on port C.** Only ports A and B can wake the chip from EM2/EM3, so neither pin can wake it. The firmware wakes on a timer and then reads the pins, or reads DRDY over I2C.
2. **The I2C bus is on port C.** I2C0 only routes to ports A/B, so use **I2C1**. It runs in EM0/EM1 only, which is fine because ADC traffic happens while awake. **VERIFY** the routing in the BG24 datasheet's alternate-function table.
3. **No LFXO.** Run the sleep timer and the Bluetooth sleep clock from **LFRCO in precision mode**, which is calibrated against the HFXO and meets BLE's 500 ppm. **VERIFY** the clock-manager setting name in SDK 2026.6.1.
4. **ADC excitation measurement.** BRIDGE_E+ and BRIDGE_E− go to ADS122C04 **AIN2/AIN3**. REFP/REFN are not connected. So the ADC uses its **internal 2.048 V reference**, measures both the bridge output (AIN0–AIN1) and the excitation (AIN2–AIN3), and the strain math divides one by the other (ratiometric in software).
   - Excitation ≈ 3.3 V × 120 / (1.8 k + 120 + 1.8 k) ≈ **0.106 V** with 1.8 k series resistors (R3, R4). Bridge current is about **0.9 mA** whenever MEASURE_EN is on.
   - Full-scale strain signal ≈ 0.106 × GF × ε / 4. That is about **53 µV per 1000 µε**, so use PGA gain 128.
5. **ADS122C04 I2C address 0x40.** A0 and A1 are both tied to GND, and RESET is tied to V_MEASURE.
6. **Back-powering risk.** I2C pull-ups go to V_MEASURE and flash pull-ups go to V_FLASH. When a load switch is off, a GPIO driven high feeds that chip through its ESD diodes. **Rule:** set a peripheral's pins to *disabled* (high-Z) before turning its switch off. Re-enable the pins only after the switch is on.
7. **The debug header J5 uses the ARM Cortex 10-pin pinout,** not Silicon Labs Mini-Simplicity. The Pro Kit's debug-out port is Mini-Simplicity, so programming the board from the kit needs an adapter or jumper wires. Pin 1 of J5 is V_SYS, so the kit's VAEM can power the board for AEM energy captures. **VERIFY** before first power-up. See [§9](#9-risks-and-open-questions).

---

## 3. Software stack

The repo is a fork of the course starter `cchoi22915/ECEN5833-xg24-starter`. Credit it as the honor code requires. What we keep from it:

- Toolchain pinned by the starter: Simplicity SDK 2026.6.1, GCC 14.2.1, SLT 1.2.1, uv-managed Python. Build and debug from VS Code.
- `tools/firmware.py` for build, flash and doctor. `tools/energy_profile.py` for AEM capture and plots. `tools/energy_budget.py` for the energy model.
- The event-driven main-loop pattern. Interrupts and timers only post events, and `app_process_action()` does the work. Sleep is managed by `power_manager`, which reaches EM2 whenever nothing is pending.
- Host tests with SDK mocks in CI (`tests/`).

What we add:

- A **RevB board target**. The project uses the bare device `EFR32BG24B210F1024IM48` instead of the `brd4186c`/`brd4002a` components, plus our own pin, clock and DCDC config. The starter's kit demo is excluded from this target.
- Silicon Labs **Bluetooth LE stack** components: `bluetooth_stack`, `gatt_configuration`, `bluetooth_feature_connection`, `_advertiser`, `_gatt_server`, `_sm` (just-works), `_power_control`.
- SEGGER **RTT logging** over SWD, compiled out in the power-measurement build.
- A `web/` folder with the laptop UI.

**No RTOS in version 1.** A main loop plus timers is enough at 10 Hz, and it's easier to reason about for sleep current. RTOS is listed as an upgrade.

---

## 4. Firmware modules

```
firmware/
  include/  board_revb.h      pin/port constants from §2.1 (single source of truth)
            app_config.h      all tunables (§1 table + thresholds below)
  src/      main.c, app.c     starter entry + event loop / state machine (§5)
            load_switch.c     MEASURE_EN / FLASH_EN with the back-power rule (§2.3-6)
            ads122c04.c       I2C1 driver: reset, config, single-shot, read
            strain.c          sample sequencing: power bridge → convert → store raw
            w25q.c            SPI flash driver: JEDEC ID, read, page program, 4 KB erase, status poll
            log_store.c       ring buffer of 4 KB blocks in flash (§6)
            energy.c          IADC CAP_SENSE + PGOOD + low-energy policy (§5.3)
            ble.c             advertising, connection, GATT handlers, transfer engine (§7)
            timebase.c        ms uptime + laptop-supplied epoch offset
            diag.c            counters + RTT logging
web/        index.html, app.js, style.css   (§8)
tests/      host tests for log_store, strain math, state machine, block encode/decode
```

Each driver has an interface that compiles on the host, so it can be unit-tested with mocks the way the starter tests `app.c`.

### 4.1 ADS122C04 settings (first pass)

| Setting | Bridge output | Excitation |
|---|---|---|
| MUX | AIN0–AIN1 | AIN2–AIN3 |
| Gain / PGA | 128, PGA on | 16, PGA on (0.106 V × 16 ≈ 1.7 V < 2.048 V) |
| Reference | Internal 2.048 V | Internal 2.048 V |
| Data rate | 90 SPS normal mode (about 11 ms per conversion; tunable) | Same |
| Conversion mode | Single-shot | Single-shot |
| Data ready | Wait the expected time in EM2, then check the DRDY pin or the DRDY bit in config reg 2 | Same |

- Measure the excitation **once per second** (every 10th sample), not on every sample. It only moves when V_SYS droops, so this halves the ADC on-time. It is tunable.
- **VERIFY** the PGA input common-mode limits at gain 128. Bridge common mode is about 1.65 V on a 3.3 V supply, which should be fine. Recheck at low V_SYS.
- **VERIFY** the ADC's power-up time before the first I2C transaction after MEASURE_EN goes high. Budget 1 ms until it is measured.

### 4.2 One strain sample (every 100 ms)

1. The timer fires and posts `EV_SAMPLE`.
2. `load_switch_on(MEASURE)`: turn on the switch, wait the settle time, then enable the I2C1 pins.
3. Write the four config registers. The ADC loses them when its power is switched off.
4. Send START, sleep for the conversion time, then read 24 bits with RDATA. Every 10th sample, also convert the AIN2–AIN3 excitation.
5. Disable the I2C pins (high-Z), then turn the switch off.
6. Append the raw code to the RAM block (§6).

Expected awake time is about 15 ms per sample. Measure it with AEM. If the switch-on overhead dominates, try keeping V_MEASURE on through a 1 s burst of 10 samples.

---

## 5. Application state machine

```
          ┌──────────┐ power-on / reset
          │   BOOT   │ init clocks, DCDC, drivers; flash: read JEDEC ID, find newest block
          └────┬─────┘
               ▼
          ┌──────────┐ 10 Hz samples into RAM block; every 1 s read CAP_SENSE + PGOOD
   ┌────► │  SAMPLE  │──── 60 s elapsed ────┐
   │      └──────────┘                      ▼
   │                                  ┌──────────┐ write block to flash (FLASH_EN on → program → off)
   │                                  │  COMMIT  │
   │                                  └────┬─────┘
   │                                       ▼
   │                                  ┌──────────┐ if connected: send unsent blocks
   └──── done / timeout / low energy ─│  STREAM  │ if not: advertise up to 10 s
                                      └──────────┘
          LOW_ENERGY: V_SYS below threshold → stop sampling + radio; wake every 10 s to recheck
```

### 5.1 Sampling and streaming together

The meeting described two separate modes. Two choices for version 1:

- **A (simplest).** Sampling pauses during STREAM, which lasts at most about 15 s. This leaves a timestamped gap every minute.
- **B.** Sampling keeps running during STREAM, since the radio and the ADC are both event-driven. The code is barely more complex.

**Proposal: B**, because data gaps hurt the demo more than a little extra code does. **Decide at review.**

### 5.2 Staying connected instead of re-advertising

Chrome's Web Bluetooth needs a user click to connect, and reconnecting by itself is awkward. Instead:

- After the laptop connects once, the board **keeps the connection**. It asks for a long connection interval while idle (about 1 s, with peripheral latency) to keep the energy cost low.
- STREAM then just sends data over the existing connection.
- The board only advertises (100 ms interval, up to 10 s per cycle) while it is disconnected.

Windows may refuse long intervals. **Measure** what Windows accepts, and fall back to about 100–200 ms if needed.

### 5.3 Low-energy policy (thresholds in `app_config.h`, to be tuned on the board)

| V_SYS | Behavior |
|---|---|
| ≥ 3.1 V (PGOOD high) | Normal |
| 2.4 – 3.1 V | Keep sampling. Skip STREAM unless already connected. |
| 1.9 – 2.4 V | LOW_ENERGY: save the partial block to flash, stop the radio and sampling, wake every 10 s |
| < 1.9 V | Let the brown-out detector reset the chip. On boot, wait in LOW_ENERGY until V_SYS > 2.6 V. |

---

## 6. Flash log format

- The W25Q128 is 16 MB = 4096 sectors of 4 KB. **One 60 s window = one sector**, so a sector is erased right before it is reused.
- Capacity: 4096 windows × 1 min ≈ **68 hours**, written as a ring buffer.
- The flash is powered only during COMMIT: once a minute, about 10–40 ms per write, plus a 4 KB erase of about 45 ms typical. **VERIFY** the timings.
- Before turning the flash off: send Deep Power-Down, set the SPI and CS pins to high-Z, then turn FLASH_EN off.

**Block layout (little-endian):**

| Offset | Size | Field |
|---|---|---|
| 0 | 4 | magic `0x44534431` ("DSD1") |
| 4 | 4 | sequence number (monotonic, never reused) |
| 8 | 8 | start time: epoch ms if the laptop has set the time, otherwise uptime ms (flag bit 0) |
| 16 | 2 | sample period ms (100) |
| 18 | 2 | sample count (≤ 600) |
| 20 | 2 | flags: bit0 time is epoch; bit1 partial block; bit2 low-energy entry; bit3 ADC error seen |
| 22 | 2 | excitation code count (≤ 60) |
| 24 | 2 × 4 | V_SYS mV: min, max, first, last over the window |
| 32 | 4 | PGOOD-low seconds in window, diagnostic counters |
| 36 | 3 × 60 | excitation codes (int24, 1 per s) |
| 216 | 3 × 600 | bridge codes (int24, 10 per s) |
| 2016 | … | padding with 0xFF |
| 4092 | 4 | CRC-32 of bytes 0–4091 |

- On boot, scan the 4096 sector headers to find the newest valid sequence number.
- "Sent" status is tracked in RAM as the highest acknowledged sequence number. It is also saved in a small flash record every 10 acks, so a reset doesn't resend everything.

---

## 7. Bluetooth GATT design

Device name: `DSD-xxxx`, where xxxx is the last 4 hex digits of the BLE address. TX power 0 dBm, adjustable.

Custom service UUID `d3bb0000-302e-4793-b606-e5f9ec21bc9b`. Characteristic UUIDs replace `0000` with the ID shown below.

| ID | Name | Props | Payload |
|---|---|---|---|
| 0001 | Status | read, notify | fw version, state, V_SYS mV, PGOOD, uptime s, newest seq, oldest seq, last acked seq, error counters |
| 0002 | Control | write | command byte + args: `SET_TIME(epoch_ms)`, `START_DUMP(from_seq)`, `ACK(seq)`, `STOP_DUMP`, `ERASE_ALL` (needs a confirm code), `LIVE(on/off)` |
| 0003 | Data | notify | block chunks: `[seq u32][chunk_index u16][chunk_count u16][bytes…]`, up to MTU−3 |
| 0004 | Live | notify | the latest bridge code + V_SYS once per second while LIVE is on (for the demo) |

**Transfer flow:**

1. The web page connects and subscribes.
2. It writes `SET_TIME` and then `START_DUMP(last_seq_it_has + 1)`.
3. The board sends each block as chunks.
4. The page checks the CRC and writes `ACK(seq)`.
5. The board moves on to the next block, or waits until the next COMMIT.

Ask for a 247-byte MTU and the 2M PHY where available. At about 244-byte chunks, one block is 17 notifications.

---

## 8. Laptop web page (`web/`)

- Plain HTML/JS with no build step. Serve it with `python -m http.server` from `web/`. Web Bluetooth needs localhost or HTTPS, and Chrome or Edge.
- It shows: a connect button; live status (V_SYS, PGOOD, state); a rolling strain chart (one small vendored chart library, or a hand-drawn canvas); the block list; and a **CSV download** of all received data.
- **Strain conversion on the laptop** (quarter bridge):
  - Vr = (Vout_code / G_out) / (Vex_code / G_ex) − Vr₀
  - ε = −4·Vr / (GF·(1 + 2·Vr))
  - Vr₀ is the zero offset, set by a "Zero" button with the axle unloaded.
- Received blocks are stored in the browser's IndexedDB, so a page reload doesn't lose data.

---

## 9. Risks and open questions

| # | Item | Plan |
|---|---|---|
| 1 | J5 is Cortex-10, the Pro Kit is Mini-Simplicity | Make or buy an adapter, or hand-wire it with 1.27 mm jumpers. Confirm the pin mapping before first power-up. |
| 2 | Boards may arrive late (week 13) | Develop drivers on the Pro Kit with jumper-wired breakouts where possible. Keep the board config in one file so switching targets is a one-line change. |
| 3 | Windows BLE connection-interval limits | Measure early. Fall back to re-advertising if long intervals are refused. |
| 4 | HFXO CTUNE for the 39 MHz crystal | Start from the BRD4186C value, then trim with Commander or an RF frequency check. **VERIFY** the crystal's CL. |
| 5 | Supercap leakage and divider current vs. harvest | Measure the sleep current with AEM. Tune the §5.3 thresholds. |
| 6 | PAVDD runs from the DCDC, which caps TX power | 0 dBm is plenty for a laptop at demo range. |
| 7 | Sampling during STREAM (§5.1 A vs B) | Decide at review. |

---

## 10. Tasks

No owners. Pick tasks at the review. Milestone order matters, but the tasks inside each milestone can run in parallel.

**M0 — Toolchain**
- [ ] Everyone: run *Starter: first setup* and build the demo
- [ ] Add the RevB target (bare device, no kit components, DCDC + HFXO + LFRCO precision config); confirm it builds
- [ ] `board_revb.h` from §2.1
- [ ] RTT logging plus `diag` counters; a power build with logging compiled out

**M1 — Bring-up on RevB** (when boards arrive)
- [ ] Power checks from L18: V_SYS-to-GND shorts, then bench 3.3 V through J4/J3
- [ ] Debug adapter for J5; flash a "hello over RTT" image
- [ ] Toggle MEASURE_EN and FLASH_EN and measure V_MEASURE and V_FLASH; confirm no back-powering with the switches off
- [ ] Read the W25Q128 JEDEC ID; read an ADS122C04 register; read CAP_SENSE through the IADC

**M2 — Drivers**
- [ ] `load_switch.c` with the pin-state rule
- [ ] `ads122c04.c` (I2C1, config, single-shot, DRDY by timer + pin/register)
- [ ] `w25q.c` (read, program, erase, deep power-down)
- [ ] `energy.c` (IADC on PB00, PGOOD read)
- [ ] Host tests for driver command encoding

**M3 — Application**
- [ ] `strain.c` sample sequencing
- [ ] `log_store.c` ring buffer, block encode/CRC, boot scan, ack record; host tests including wrap-around and power-loss mid-write
- [ ] State machine in `app.c` (§5), including LOW_ENERGY
- [ ] `timebase.c`

**M4 — Bluetooth + web**
- [ ] GATT database (§7) and `ble.c` advertising/connection/notify engine
- [ ] Transfer protocol with acks; connection-parameter policy (§5.2)
- [ ] `web/`: connect, status, live chart, dump + IndexedDB, CSV export, Zero button
- [ ] End-to-end test: 1 h log, disconnect/reconnect mid-dump, reset mid-dump

**M5 — Energy and verification**
- [ ] AEM captures: sleep floor, one sample, COMMIT, STREAM block, advertising window
- [ ] Energy budget JSON from the measurements (`tools/energy_budget.py`); compare to harvest at the demo source
- [ ] Shunt-resistor test on harvested power (meeting plan)
- [ ] Tune §5.3 thresholds and data rate

**M6 — Dev-kit demo build (Dec 1)**
- [ ] Pro Kit target with the same app; real ADC on breakouts or a simulated source; RTT/LCD output
- [ ] Demo script

**Upgrades (after version 1 works)**
- [ ] Sample rate scaled to available energy (V_SYS slope)
- [ ] State-of-charge-based mode switching (e.g. sample through the first 75 % of charge, stream in the last 25 %)
- [ ] RTOS (FreeRTOS/Micrium via SDK) in place of the main loop
- [ ] Laptop-writable sample rate and cycle length (Control characteristic)

---

*Sources: team meeting notes 2026-11-06; ECEN 5833 lectures L01–L18; RevB PcbDoc/netlist; course starter README.*
