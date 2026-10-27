# Alarm V3: design decisions and gotchas

Archive notes for the ESP32-C6 alarm clock, written 27 October 2026 once the project was finished and delivered. They cover the final firmware in `ESP32C6 Feather/` (PlatformIO) and `Alarm client.X/` (MPLAB X, PIC16F15225). Where these notes and older chat notes disagree, the source code wins: these notes were written from the code.

---

## 1. Hardware summary

| Function | Part | Notes |
|---|---|---|
| Main MCU | Adafruit ESP32-C6 Feather #5933 | RISC-V. On-board MAX17048 fuel gauge. IO20 switches VSENSOR, which powers the I2C pull-ups, NeoPixel and STEMMA QT |
| Companion MCU | PIC16F15225, PDIP-14, 3.3 V | I2C client at 0x08. Handles every output that has to stay on or be timed while the ESP32 is in deep sleep |
| RTC | PCF8563T on an HW-689 breakout | Board cut down, coin cell holder and 10 µF electrolytic removed. Runs from the 3.3 V rail, no backup cell |
| Display | GoodDisplay GDEY029T94-FL03 (SSD1680), 2.9", front-lit | Connected through a DESPI-C02 adapter. Front-light LEDs driven straight from a PIC pin |
| Audio | MAX98357A I2S Class-D breakout, 4 Ω 3 W 50 mm speaker (reclaimed) | Amp powered from the LiPo. 560 kΩ added between IO15 and SD. The design calls for R1 (1 MΩ SD_MODE pull-up to VIN) to be removed. **As built, R1 is still fitted, which was an oversight (see gotcha 25).** 220 µF across VIN/GND. Speaker in an enclosure |
| Alarm-time input | Two 12-position rotary switches on resistor ladders | Ladders powered from VSENSOR. Hour dial on IO1, minute dial on IO4 |
| Button | Top switch on IO6 | Active low, wakes the chip from deep sleep |
| Battery | 2000 mAh LiPo | |
| PDLC film | Separate stripboard sub-board | 4047 oscillator, 2N7000 push-pull into the centre tap of an LT44 transformer |
| Moomin light | Existing touch-operated light | PIC pulses an opto-coupler for about 500 ms to simulate a touch |
| Programming | Home-built FTDI-based board, UART0 (IO16/IO17) | Standard RTS/DTR two-transistor auto-reset on RST and IO9 |

The build uses breakout boards and through-hole parts on stripboard. A PCB was considered and dropped.

### ESP32-C6 pin map (from `Constants.h`)

| IO | Use |
|---|---|
| 0 | I2S BCLK |
| 1 | Hour dial ladder (ADC) |
| 2 | eInk CS |
| 3 | eInk SCK |
| 4 | Minute dial ladder (ADC) |
| 5 | eInk SDI (MOSI) |
| 6 | Top button (ext1 wake) |
| 7 | PCF8563 INT (ext1 wake) |
| 8 | I2S LRCLK. This is also a strapping pin, so it has an external pull-up (see gotchas) |
| 9 | BOOT (auto-reset circuit) |
| 14 | I2S DIN |
| 15 | MAX98357A SD_MODE through 560 kΩ. The on-board red LED lights while the amp is enabled |
| 16 / 17 | UART0 RX/TX, used for flashing and the serial monitor |
| 18 / 19 | I2C SCL/SDA, shared by the PCF8563 (0x51), MAX17048 (0x36) and PIC (0x08) |
| 20 | VSENSOR / I2C pull-up power. Must be high when awake |
| 21 | eInk D/C |
| 22 | eInk RES# |
| 23 | eInk BUSY |

### PIC16F15225 pin map (from `main.c`)

| Pin | Use |
|---|---|
| RA5 | eInk front light (CCP1 PWM fade, 100 Ω series resistor, 5.2 mA measured) |
| RA2 | Moomin opto-coupler pulse |
| RC2 | PDLC sub-board ENABLE (the sub-board has its own pull-down) |
| RC0 / RC1 | I2C SCL / SDA (MSSP1 through PPS) |
| RA0, RA1, RA4, RC3–RC5 | Unused. Driven low as outputs to cut floating-input current in sleep |

---

## 2. Firmware architecture (ESP32-C6)

### Boot-to-sleep model
- `setup()` always finishes in `esp_deep_sleep_start()`, and `loop()` never runs. Every wake, from the RTC tick or a button press, is a fresh boot.
- The only state kept across wakes is in `RTC_DATA_ATTR` variables:
  - `lastWakeUtc` (Alarm)
  - `updatesSinceFullRefresh` (Display)
  - `nextSyncUtc`, `retriesLeft` and `lastSuccessUtc` (TimeSync)

  All of these reset on a cold boot.
- The wake cause is read with `esp_sleep_get_wakeup_cause()`. Anything other than EXT1 counts as a cold boot and runs `powerOnInit()`.

### Module convention
Every module exposes these three functions, even when one does nothing:
- `init()` runs on every boot, including deep-sleep wakes. It sets up ESP32-side tasks, pins and interrupts.
- `powerOnInit()` runs on a cold boot only. It does one-time configuration of external hardware (RTC, gauge, panel).
- `prepareForSleep()` puts the module's pins in their sleep state and returns its ext1 wake-mask bits. `main.cpp` ORs the masks together and calls `esp_sleep_enable_ext1_wakeup(mask, ANY_LOW)`.

All other functionality goes in its own explicit function. The startup order is a deliberate dependency chain, and it is documented in `main.cpp`:
1. Display before `powerOnInit()`.
2. Music and AlarmSequence before Clock and Button, because those tasks can call them as soon as their interrupts attach.

### Single owners
- **Clock** is the only interface to the PCF8563. Nothing else includes `rtc/pcf8563_timer.h` or touches `RTC_INTERRUPT_PIN`.
- **Battery** is the only interface to the MAX17048.
- **TimeSync** owns the sync schedule and the Wi-Fi connect and disconnect. **Clock** owns the NTP exchange and the RTC write.
- Once the tasks are running, every RTC access happens on the clock task, including the daily NTP resync. RTC transactions therefore never interleave with each other.
- **Wireless** has that name because a project header called `Network.h` would shadow the arduino-esp32 core's own `Network.h`, and the core already defines a global called `Network`.

### Tasks and signalling
- Each interrupt-driven module has an ISR that does three things: disables its own interrupt, calls `Sleep::inhibit()` and sets an event bit. A task then does the real work, re-enables the interrupt and calls `Sleep::release()`.
- Interrupts are `ONLOW` (level). The ISR disables itself, and the pin is re-armed only after the cause has been cleared: TF/AF cleared for the RTC, or a debounced release for the button.
- Event bits come from one shared event group, allocated in sequence by `Utils::getNextEventBit()`.

### Sleep gate (`Sleep.cpp`)
- Sleep is blocked by a reference count. Anything with pending work calls `inhibit()` before the work becomes pending and `release()` when it is done. `setup()` blocks in `waitUntilAllowed()` until the count is zero.
- The final implementation is a `std::atomic<int>` plus a direct-to-task notification:
  - The C6's RV32IMAC core has the A extension (`amoadd.w`), so the atomic is lock-free and safe to use from an ISR.
  - The count is the authority. The notification only wakes the waiter so it checks the count again.
  - This replaced the earlier plan of a `portMUX` critical section plus an event-group bit.
- A FreeRTOS counting semaphore was rejected because it can't express "wait until the count is zero".

### I2C bus (`I2CBus.cpp`)
- `TwoWire` holds its own mutex for each transaction, but not while `Wire.available()`/`Wire.read()` run after `requestFrom()`. Another task's transfer could overwrite the receive buffer in that gap.
- `I2CBus::Guard` is a scoped, statically allocated FreeRTOS mutex. It covers the whole transaction, including reading the bytes out.
- `pcf8563SetDateTime()` holds one Guard across the whole STOP-to-release sequence, so a PIC command can't land inside the timed write.
- Each MAX17048 register read is one self-contained transaction under its own Guard.

---

## 3. Timekeeping

### Time representation
- The RTC and all internal times are **UTC**. Local time is used only for display and for interpreting the alarm dials.
- The time zone is a POSIX TZ string applied with `setenv`/`tzset`. No timezone library is used.
  - The value is `GMT0BST,M3.5.0/1,M10.5.0`, hard-coded until there's a configuration UI.
  - TZ lives in RAM, so it is set again on every boot, including wakes.
- `timegm()` isn't standard C, so `Clock::readUtc()` converts dates with Howard Hinnant's `days_from_civil`.

### RTC choice and power
- **PCF8563** was chosen over the DS3231 for its quiescent current (about 0.25 µA).
- The bare chip is SMD-only: NXP dropped the DIP-8 package in 2015. That is why a breakout is used.
- The RTC runs from the 3.3 V rail with no coin cell. A backup cell adds nothing here because the clock resyncs daily and the VL flag catches any power loss.
- CLKOUT is disabled at power-on. Its default is 32.768 kHz, which costs about 0.3 µA extra.

### Tick source: minute alarm, not countdown timer
- Normal use (`TICK_ON_THE_MINUTE = true`): the **minute alarm** asserts INT at hh:mm:00, so the display and the alarm check change exactly on the minute.
- The countdown timer's source clock has no documented phase relationship to the time counters, so it can't be aligned to the minute. It is kept as a test mode with a 10 s free-running tick (`TEST_TICK_PERIOD_S`).
- INT is **level** mode: it stays low until the flag is cleared. ESP32-C6 LP-IO wakeup needs this.
- Because the alarm matches only one minute value, it is moved on every tick (`pcf8563ClearAlarmFlag()` then `pcf8563ArmMinuteAlarm()`). It is re-armed after every time-set as well.
- The clock task acknowledges the tick before reading the time. If the minute rolls over in between, the display is never left a minute behind.

### NTP: own UDP exchange, not the SNTP client
- `configTime()`/lwIP SNTP was rejected for two reasons:
  - It waits a random 0–5 s before its first request (`CONFIG_LWIP_SNTP_STARTUP_DELAY`).
  - It retries only after lwIP's 15 s `SNTP_RECV_TIMEOUT`.
- `Clock::syncNetworkTime()` sends up to three requests, 1 s apart, then waits a final 3 s.
  - Each request goes to a different pool hostname (`0/1/2.uk.pool.ntp.org`). lwIP caches DNS for the TTL and NetworkUDP uses only the first address, so retrying one name would keep hitting the same server.
  - All requests share one socket, so a late reply to an earlier request is still accepted.
- Each request carries a random transmit timestamp as a nonce. Replies are rejected on any of:
  - kiss-o'-death
  - wrong mode
  - LI = 3
  - stratum > 15
  - no matching nonce
- Half the round trip is added to the server's transmit time.
- **The RTC write is aligned to the second.** The PCF8563's first seconds increment comes 0.507813–0.507935 s after STOP is released (datasheet §8.10, Table 26).
  - The write is timed so that STOP releases 507.874 ms before a true second boundary.
  - It starts a further 3.2 ms early, the measured duration of the nine-register write.
  - The firmware sleeps for most of the wait and spins for the last 2 ms.
- Modem sleep is disabled after connecting (`WiFi.setSleep(false)`). Otherwise the station wakes only at DTIM beacons, which can delay the reply by up to the AP's DTIM period.

### Sync schedule (`TimeSync.cpp`)
- The clock syncs at cold boot, then daily at **03:00 UTC**. That is 03:00 GMT or 04:00 BST, clear of the UK clock changes at 01:00 UTC.
- After a failed sync it retries on each of the next **10** minute ticks. If all of those fail it waits for the next 03:00; the RTC keeps good enough time for another day.
- A sync that completes up to 60 s before a scheduled time counts as that slot's sync (`SLOT_TOLERANCE_S`). This covers a fast RTC firing the tick early.
- The sync runs after the display update and the alarm check, so Wi-Fi never delays them. It still runs under the tick's sleep inhibit.
- If the RTC was invalid (VL set) and the sync fixed it, the display is redrawn straight away.
- Sync times are UTC slots and a few seconds of drift between syncs is acceptable, so no drift compensation is done.

---

## 4. Alarm

### Dials
- Each 12-position switch taps an equal-step ladder from 0 V to VSENSOR, about 300 mV per step. Each reading averages 8 ADC samples (`analogReadMilliVolts`).
- A reading more than **100 mV** from any step is rejected, and the tick is skipped without recording it. This tolerance comes from two error sources:
  - ADC calibrated error at 12 dB attenuation is ±40 mV (C6 datasheet Table 5-6).
  - VSENSOR's regulator tolerance adds to that, because the ADC isn't ratiometric to the ladder supply.
  - A `static_assert` checks that the tolerance stays under half a step.
- The hour dial reads 0–11 and the minute dial 00–55 in 5-minute steps. **0:00 means the alarm is off.**
- The 0–11 hour dial was kept. The alarm fires at both the dial time and 12 hours later, for example 8:05 gives 08:05 and 20:05.

### Due logic (`Alarm::isDue`)
- The alarm is due if either local occurrence of the dial time falls in the interval (`lastWakeUtc`, now]. It checks windows rather than comparing for equality, so missed ticks, DST changes and midnight are all handled correctly.
- `mktime()` with `tm_isdst = -1` works out DST for the day in question.
- The alarm never fires on the first tick after a cold boot (there's no last wake yet) or while VL is set.

### Alarm sequence (`AlarmSequence.cpp`)
The sequence is a three-stage state machine (Idle → Ringing → FilmClear), with every transition made under one mutex.

| Event | Action |
|---|---|
| Alarm due | Music starts, Moomin is pulsed once, the PDLC film clears, and the front light is turned on and held |
| 1st press (Ringing) | Music stops. The film stays clear |
| No press, music ends | Film and front light off, back to Idle |
| 2nd press (FilmClear) | Film and front light off |
| No 2nd press within 30 s | Film and front light off |

- The Moomin light is never turned off by the firmware. Its only control is a touch pulse, and its state can't be read back.
- The sequence holds a sleep inhibit from `start()` until `finish()`. Every path ends with the film and light off and the inhibit released.
- When no alarm is running, a button press turns the front light on for the PIC's on-time.

---

## 5. Display

- **Layout:**
  - 12-hour H:MM time with no seconds, as large as fits on the left, placed a little below centre.
  - AM/PM in a narrow right-hand column on the time's baseline.
  - A phone-style battery indicator at the top left. The percentage is XOR-drawn over the fill so it reads on both black and white.
  - A 6 px margin on all sides, because the panel mount covers the edge of the active area.
- **Fonts:** Liberation Sans Bold at three sizes, generated by `tools/make_gfx_font.py`. The script uses Pillow monochrome rendering and outputs Adafruit GFX format.
  - The digit size is the largest at which the widest possible time, "14:44", fits in 237 px.
  - The colon is vertically centred on the digits.
  - A one-digit hour is shifted right so the colon and minutes never move.
- **Refresh strategy:**
  - The time is always drawn with a partial refresh (BUSY about 0.57 s).
  - Every 10th update first does a full refresh to plain white (BUSY about 4.3 s) to clear ghosting, then draws with a partial.
  - The cold boot does the same full-white refresh, which seeds both RAM planes so partial refreshes work.
- The panel enters SSD1680 deep sleep **mode 1** after every update. Mode 1 keeps both RAM planes, so the next partial refresh after an ESP32 wake has the previous image to work against.
- The vendor driver's `Epaper_READBUSY()` spin-waits. It was replaced with a `vTaskDelay`-based poll (`waitWhileBusy`) so other tasks keep running during refreshes.
  - Timeouts are 2 s for a partial and 10 s for a full refresh. They exist only so a faulty panel can't hold off sleep forever.
- The image is drawn on a `GFXcanvas1` in landscape, then remapped into the native 128×296 portrait buffer using `EPD_ROTATION = 3` and `EPD_FLIP_H = 1`.

---

## 6. Audio

- An I2S Class-D amp was chosen over PWM plus an RC filter to keep the component count and board area down.
- **Library and file:**
  - BackgroundAudio (earlephilhower) decodes MP3 from LittleFS, played through a `PhilipsI2SAudio` subclass of `ESP32I2SAudio` (see gotchas).
  - The file is Grieg's "Morning" from *Peer Gynt*, from the Musopen/Wikimedia Commons recording, which is public domain.
- **Encoding target:**
  - About 22.05 kHz, mono, 56–64 kbps CBR.
  - Low-pass at about 7–8 kHz, plus a high-pass to tame the timpani.
  - Trimmed to length.
  - Always re-encoded from the original source MP3, never from the device file.
- **Gain** is 0.2 in the decoder, set by ear.
  - That gives 0.72 Vrms, about 0.13 W into 4 Ω: full scale is 3.59 Vrms × gain at the amp's default 9 dB.
  - 1 % THD+N would be reached at a gain of about 0.64.
- Three changes together cured the bass distortion:
  - the 220 µF capacitor on the amp's VIN
  - the speaker enclosure
  - the high-passed re-encode
- **Amp shutdown:** IO15 drives SD_MODE through an added 560 kΩ resistor. The SD node also has the breakout's R1 (1 MΩ to VIN) and SD_MODE's internal 100 kΩ (92–108 kΩ) pull-down.
  - The 560 kΩ resistor forms a divider, so that IO15 high puts SD in the (L/2 + R/2) band rather than at full logic high, which would select the left channel only. For reference, the datasheet's R_LARGE for 3.3 V drive is 634 kΩ (Table 6).
  - The resulting voltages, and a possible problem with shutdown in sleep, are worked through in gotcha 25. **Note that the comment in `Music.cpp` gives 0.46–0.53 V and says the pull-down alone holds the amp in shutdown during deep sleep. That calculation leaves out R1.**
- **Power sequencing:**
  - The amp is enabled before I2S starts. The silence pre-loaded into the DMA buffers covers its 7.5 ms turn-on.
  - The amp is disabled before I2S stops.
  - After end of file, playback waits 300 ms for the DMA buffers to drain before stopping.
- LittleFS is mounted with `formatOnFail = false`, so a failed mount can't wipe the uploaded music.

---

## 7. Battery gauge

- The indicator shows the MAX17048 ModelGauge **state of charge** as a phone-style percentage, always visible. There is no separate low-battery warning.
  - It uses the generic built-in model, not one tuned to this cell. The SOC accuracy is doubtful: it once read 13 % at 3736 mV.
- The gauge is forced into **permanent hibernate** (`HIBRT = 0xFFFF`). Hibernate draws 3–5 µA against 23–40 µA active, and the gauge converts every 45 s.
- The **reset comparator** is disabled (`VRESET.Dis`), saving about 0.5–1 µA. Cell removal is still detected, just more slowly.
- **EnSleep and CONFIG.SLEEP** are explicitly cleared. Otherwise SDA/SCL sitting low through deep sleep could put the gauge into sleep mode (see gotchas).

---

## 8. PIC16F15225 companion

### Role
The PIC handles everything that has to keep running, or be timed, while the ESP32 is in deep sleep:
- the front-light on-time and fades
- the Moomin pulse
- holding the PDLC enable

The ESP32 can then send one I2C byte and go straight back to sleep.

### Protocol
Each command is a single byte written to address 0x08. The values must match on both sides.

| Byte | Command |
|---|---|
| 0x01 | Front light on, then off after the PIC's on-time (4 s nominal, 3.4–4.7 s actual). A repeat ON restarts the timer |
| 0x02 | Front light off |
| 0x03 / 0x04 | PDLC enable on / off |
| 0x05 | Moomin pulse (500 ms nominal, 430–585 ms actual) |
| 0x06 | Front light on with no time limit (used while the alarm is active) |

### Implementation
- The clock is HFINTOSC at 4 MHz for low active current. A 100 kHz I2C client needs FOSC ≥ 1.5 MHz.
- The PIC sleeps between packets:
  - The MSSP wakes it with `SSP1IE` plus `PEIE`. `GIE` is deliberately left off, so execution just continues after `SLEEP()` and the code stays a simple polling loop.
  - Once an address has matched, it busy-waits for the data byte instead of sleeping again.
- The front-light fade uses CCP1 PWM at 244 Hz (Timer2, 10-bit) and takes about 0.26 s.
  - The duty follows level²/16, a rough match to perceived brightness.
  - Between fades RA5 is a plain LAT output, so Timer2 stops and the PIC can sleep.
- The Moomin pulse is timed by Timer0 and the front-light on-time by Timer1. Both are clocked asynchronously from LFINTOSC, so they keep counting in sleep and wake the PIC when they finish.

---

## 9. PDLC sub-board

- **Final circuit:** a 4047 astable drives two 2N7000s in push-pull into the centre tap of an LT44 transformer. There is no H-bridge and no boost converter.
  - It runs from 3.7 V (LiPo), giving about 22 V RMS across the film at about 7 mA.
  - On the bench the film still cleared at 28 Vp-p (2 V supply).
- **Inputs:** LiPo+, ground and ENABLE from the PIC. That is three wires; four wires was ruled out. ENABLE has its own pull-down on the sub-board.
- **Supply rules:**
  - The film is not driven from the MCUs' regulated rail.
  - A separate battery was rejected (a 9 V option was tried).
- **Designs tried on the way:**

  | Driver | Supply | Result |
  |---|---|---|
  | DRV8871 | 9 V | About 60 Vp-p. 4047 timing R of 220 kΩ gave about 106 Hz, 30 V RMS at 7 mA |
  | DRV8833 | 3.7 V | Drove a 230 V:18 V transformer |
  | DRV8833 | 3.7 V | Drove the LT44 centre tap |

  Minimum useful drive with the H-bridge was about 40 Vp-p.
- The film is only powered while the alarm sequence is active.

---

## 10. Power and measurements

| Condition | Measured |
|---|---|
| Whole board in deep sleep (ESP32, PIC, RTC, eInk and driver) | 2.5 µA |
| Minute wake with partial refresh | About 20.6 mC over about 0.8 s, about 26 mA average |
| Share of that spent with the ESP idling (about 23 mA) waiting for eInk BUSY | About 60 % |
| Partial refresh BUSY | About 0.57 s |
| Full refresh BUSY | About 4.3 s |
| eInk interface pins left floating in sleep (before the fix) | About 0.8 mA |
| Front-light LED current | 5.2 mA |

- Partial-refresh wakes alone average about 0.34 mA (20.6 mC every 60 s). Full refreshes and the daily Wi-Fi sync add to that, but the total hasn't been measured as a whole.
- The target was about 4 months between charges on 2000 mAh.
- Sleep current is under 1 % of the charge used per minute, so active time is what matters.

---

## 11. Build and flash

- The toolchain is VSCode with PlatformIO, the pioarduino platform and the Arduino framework.
- **Flashing:** over UART0 (IO16/IO17) through the FTDI board, not USB-Serial-JTAG. This freed IO12/IO13.
  - `ARDUINO_USB_CDC_ON_BOOT=0` keeps `Serial` on UART0.
  - `post_upload_reset.py` opens and closes the port after upload. That toggles RTS/DTR, which resets the board into the new firmware.
- **Partitions** (`partitions.csv`): nvs, otadata, a single 0x1F8000 app slot and a 0x1F8000 LittleFS partition. There's no second OTA slot.
- The music is uploaded with PlatformIO's filesystem upload from `data/`.
- The vendor eInk driver is in `lib/GoodDisplay_GDEY029T94`. Its SPI header picks up the pin numbers from `src/Constants.h`.
- **Wi-Fi credentials are hard-coded in `src/Wireless.cpp`.** Keep that in mind if the source is ever shared or published.

---

## 12. Gotchas

### ESP32-C6: sleep, wake and boot
1. **ext1 wake pull-ups need `rtc_gpio_init()` first.**
   - After a power-on reset ESP-IDF gates the LP IO clock off (`LPPERI_LP_IO_CK_EN`). `rtc_gpio_pullup_en()` doesn't turn it back on, so the pull-up write is silently lost.
   - The symptom is an immediate, endless wake loop.
   - The fix: `rtc_gpio_init()` → `rtc_gpio_set_direction(INPUT_ONLY)` → `rtc_gpio_pullup_en()` → `rtc_gpio_pulldown_dis()`. A meter confirmed 3.26 V on the pin throughout sleep.
2. **`rtc_gpio_deinit()` the wake pins at boot** before `pinMode`/`attachInterrupt`. Otherwise they stay under LP IO control.
3. **Deep-sleep wake from the RTC must be level-triggered.** ISRs don't run in deep sleep. The RTC flag (AF/TF) has to be cleared before sleeping, or INT is still low and the chip wakes straight back up.
4. **eInk interface pins float in deep sleep.**
   - Floating pins draw through-current, and a floating RES# can reset the SSD1680 out of its own sleep. Sleep current rose to about 0.8 mA.
   - The fix is to drive RES#/CS# high and SCK/SDI/D/C low, then call `gpio_hold_en()`. Per-pad hold works in deep sleep on the C6, and `gpio_deep_sleep_hold_en()` doesn't exist on this chip.
   - **On wake, set each pin as an output at its held level before `gpio_hold_dis()`**, or it glitches to the pad default.
   - Call `SPI.end()` first so SCK and MOSI can be driven as GPIOs.
5. **GPIO8 is a strapping pin and floats at reset** (C6 datasheet Table 3-1). UART download mode needs GPIO8 = 1 and GPIO9 = 0 (Table 3-3).
   - Without an external pull-up, auto-download failed intermittently with "Wrong boot mode detected (0x0)".
   - GPIO8 doubles as I2S LRCLK, so the pull-up stays in place.
6. **Wi-Fi before deep sleep:**
   - `WiFi.mode(WIFI_OFF)` is needed (it calls `esp_wifi_stop()` and `esp_wifi_deinit()`).
   - `WiFi.persistent(false)` must come before anything starts the driver. Otherwise every `WiFi.begin()` writes the SSID and password to NVS.
7. **`portMAX_DELAY` blocks forever only if `INCLUDE_vTaskSuspend == 1`.** ESP-IDF hard-codes that today, and `main.cpp` has an `#error` guard in case it ever changes.

### I2C bus
8. **The Feather's I2C pull-ups are on the IO20-switched VSENSOR rail, and the HW-689 has none of its own.**
   - IO20 has to be high when awake. The board variant does this; the Arduino Adafruit package does it in `initVariant()`.
   - The PIC's weak pull-ups are deliberately not used.
   - In deep sleep SDA and SCL sit low.
9. **The MAX17048 can enter sleep mode when SDA and SCL are both held low for 1.75–2.5 s, if EnSleep is set.** In sleep it stops tracking the cell, so the firmware explicitly clears EnSleep and CONFIG.SLEEP.
10. **Wire's internal mutex doesn't cover `Wire.read()` after `requestFrom()`.** Use `I2CBus::Guard` for the whole transaction. The Guard isn't recursive, so never nest one.
11. **MAX17048 registers are 16-bit and must be read or written as a whole word in one transaction.** 8-bit writes have no effect. MODE is write-only.

### PCF8563
12. **The minute alarm fires only when the minute counter increments to a match.** If the alarm is written equal to the current minute, it waits an hour. `pcf8563ArmMinuteAlarm()` re-reads the minute after writing and retries if it rolled over.
13. **Re-arm the minute alarm after every `SetDateTime()`.** It was armed relative to the old time.
14. **Control_status_2 bits 7:5 must be written as 0.** Writing 1 to a flag bit leaves it unchanged, so clear one flag by writing 0 to it and 1 to the other.
15. **Set the time with the STOP-bit sequence.** The first seconds increment comes about 507.9 ms after STOP is released, not a full second. This is what makes the sub-second NTP alignment possible.
16. **The countdown timer isn't phase-locked to the time counters.** Don't use it for an on-the-minute tick.
17. **Check VL before trusting the time.** The alarm check is skipped while VL is set, so a bad time is never recorded as the last wake.

### eInk (GDEY029T94 / SSD1680)
18. **Seed both RAM planes with a full refresh at power-on.** Without it, partial refreshes ghost and drift against stale RAM. The same applies if the panel ever loses power.
19. **Black drawn by a full refresh faded to uneven grey about 0.5 s later.** Black drawn by a partial refresh onto a white full-refreshed screen stays solid, which is why full refreshes only ever clear to white.
    - This was seen on the first panel, which was later found to be damaged. That panel's other symptom was a horizontal line at a fixed row appearing about 30 s after each refresh. Swapping in the spare panel on the same DESPI-C02 cured it.
    - The white-then-partial approach was kept regardless.
20. **Deep sleep mode 1 (0x10 = 0x01) keeps RAM.** Mode 2 doesn't, and would force a full refresh after every ESP32 wake.
21. **Refresh times lengthen in the cold.** The specified 0.3 s and 2 s are at 25 °C; measured times are 0.57 s and 4.3 s. Allow margin in the BUSY timeouts.

### Audio
22. **The MAX98357A takes standard (Philips) I2S. The MAX98357B takes left-justified.**
    - `ESP32I2SAudio::begin()` hard-codes `I2S_STD_MSB_SLOT_DEFAULT_CONFIG` (left-justified). The A reads each word one bit late, so every sample comes out doubled (+6 dB) and anything above half scale wraps round.
    - `PhilipsI2SAudio` reconfigures the slot to Philips after `begin()`. The channel has to be disabled to change it.
23. **A filterless Class-D output relies on the speaker's coil inductance.** A small or salvaged speaker with low inductance lets hiss through.
    - The fix is a series LC output filter: 22 µH / 0.22 µF per Maxim's evaluation board, or 100 µH / 0.22 µF as a substitute.
24. **Bass distortion on the LiPo supply** was cured by a 220 µF capacitor across VIN/GND, a speaker enclosure and a high-passed re-encode, at gain 0.2.
25. **SD_MODE has three resistors on it: R1 (1 MΩ to VIN), the added 560 kΩ from IO15, and the internal 100 kΩ (92–108 kΩ) pull-down.** The trip points are B0 0.08/0.16/0.355 V and B1 0.65/0.77/0.825 V (min/typ/max). The SD voltage in each state, worked out for VIN = 3.0–4.2 V across the pull-down tolerance:

    | IO15 | SD_MODE | Result |
    |---|---|---|
    | High (3.3 V) | 0.65–0.84 V | Straddles B1, so either (L/2 + R/2) or right channel. The file is mono, so both sound the same |
    | Low | 0.22–0.35 V | **Above B0 typical (0.16 V)**, so a typical part is *not* in shutdown |
    | Floating (deep sleep) | 0.25–0.41 V | **Above B0 typical, and possibly above B0 maximum** |

    - IO15 low does not put SD at 0 V, because IO15 reaches SD only through the 560 kΩ resistor. R1 to VIN and the parallel combination of 560 kΩ and the internal pull-down form a divider, giving about 0.29 V at 3.7 V.
    - SD_MODE isn't a logic input. It feeds three comparators, and the figures above are the spread of each comparator's threshold between parts. The amp is in shutdown only when SD is below B0, so shutdown is guaranteed below 0.08 V (B0 minimum) and guaranteed not to happen above 0.355 V (B0 maximum).
    - When SD_MODE isn't in shutdown and there's no BCLK, the amp sits in **standby: 340 µA typ, 400 µA max** (datasheet: SD_MODE = 1.8 V, no BCLK, 25 °C). True shutdown is 0.6 µA.
    - If the 2.5 µA whole-board sleep figure was measured with the amp's VIN on the same supply, the amp evidently does reach shutdown on this particular part. If the amp wasn't on the measured supply, measure SD_MODE and the amp's VIN current in deep sleep: 340 µA would be about 135× the rest of the board's sleep current.
    - **The fix is to remove R1, as the design intended; it was left fitted by mistake.** With R1 removed, IO15 low gives 0 V and floating gives 0 V through the internal pull-down. IO15 high gives 0.46–0.53 V, which is in the (L/2 + R/2) band. These are the figures in the `Music.cpp` comment.

### Networking
26. **The lwIP SNTP client has a built-in 0–5 s random start delay and a 15 s retry.** Use a direct UDP NTP request when the time awake matters.
27. **lwIP caches DNS for the TTL and NetworkUDP uses only the first address.** Retry against different pool hostnames, not the same name.
28. **Modem sleep (the Arduino default `WIFI_PS_MIN_MODEM`) can delay replies by up to a DTIM period.** Disable it after connecting, because the driver reapplies the default on connect.

### PIC16F15225
29. **VDDAR must be LO (1.8–3.6 V) for 3.3 V operation.** It was HI at first.
30. **`ANSELx` resets to analog, which disables the digital input buffer, including for PPS peripheral inputs.** The I2C pins RC0 and RC1 must be set digital.
31. **To wake from sleep on an MSSP event, both `SSP1IE` and `PEIE` must be set.** `GIE` isn't needed. With GIE off, execution continues after `SLEEP()` without vectoring to an ISR.
32. **Going back to sleep between the address byte and the data byte was the suspected cause of lost data bytes.** Busy-wait mid-transaction instead.
33. **An address-only transaction (a bus probe: address, then Stop) leaves the client waiting for a data byte.** Poll `SSP1STATbits.P` to detect the Stop and allow sleep again.
34. **CCPR1 is 10 bits, so a duty of 1024 wraps to 0.** Clamp at 1023, or a fade-out from fully on blanks for one period.
35. **CCP PWM freezes in sleep.** Stay awake only while fading, and leave the pin as a plain LAT output otherwise.
36. **Write the new duty and wait for a Timer2 period match before handing the pin to CCP1 through PPS.** The duty register is double-buffered, and handing over early makes the light blink.
37. **Timers that must run in sleep need LFINTOSC with ASYNC (Timer0) or SYNC (Timer1) set.** Timer2 can't do this because PWM requires FOSC/4.
38. **LFINTOSC is 26.5–36 kHz**, so the pulse timing spreads accordingly: Moomin 430–585 ms, front light 3.4–4.7 s.
39. **TRIS resets to input, so RC2 floats during reset and programming.** The PDLC ENABLE input therefore needs its own pull-down, which is on the sub-board.

### Process
40. **Use PDFs, not PNGs, when printing dials from Mac Preview.** Preview handles PNG DPI metadata unreliably. Dials are generated in Python (Pillow + img2pdf) at 300 DPI on A4.

---

## 13. Superseded directions (for context)

- **Earlier platform:**
  - XIAO ESP32-C6, later XIAO ESP32-S3
  - RV-3028-C7 RTC
  - PDM audio through an RC filter and a 2N3904
  - A secondary SSD1306 OLED and a rotary encoder for setting the time
- **I2S pins:** IO21/22/23 were the first assignment. They became eInk pins, and I2S moved to IO0/IO8/IO14.
- **PDLC drive:** went from boost converter plus H-bridge, to H-bridge plus transformer, to the final 4047 push-pull into the LT44 centre tap.
- **Sleep gate:** a `portMUX` counter with an event-group bit was planned. It became an atomic counter with a task notification.

## 14. Deferred ideas

- **Light-sleep the ESP32 while waiting for eInk BUSY.**
  - Method: `gpio_wakeup_enable` on BUSY low + `esp_sleep_enable_gpio_wakeup` + `Serial.flush` + `esp_light_sleep_start`, inside `waitWhileBusy`.
  - It would need a light-sleep inhibit counter, held by Music and AlarmSequence, because light sleep stops all tasks and I2S.
  - Estimate: a partial wake drops from about 20.6 mC to about 7 mC, about 0.13 mA average, about 1.5 years on 2000 mAh.
  - A smaller alternative is `setCpuFrequencyMhz(80)`, which saves about 3 mA (about 10 % of the idle current).
- **Look again at the I2C locking granularity** (per-transaction Guard) if more bus users or periodic work are added.
- **A UI for configuring the time zone** (it is currently a hard-coded UK POSIX string).
- **Evaluate `esp_pm_lock`** as a native alternative to the sleep gate.
