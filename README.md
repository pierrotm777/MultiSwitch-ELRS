# RC MultiSwitch-E — ESP32-S3 Super Mini port

**English documentation — v0.2h2 (24 September 2026)**  
**Original project: WMuCpp / RC MultiSwitch-E by Wilhelm Meier.**  
**Status: partial port, functional for switching, Intervall, PWM and Morse; not the complete STM32 firmware.**

## 1. Original project, sources and license

This firmware adapts part of **Wilhelm Meier’s RC MultiSwitch-E**, from **WMuCpp**, to an **ESP32-S3 Super Mini**. The aim is to preserve CRSF MultiSwitch addressing, commands, and radio-discoverable CRSF parameters while replacing STM32 peripherals with ESP32-S3 implementations.

Original source links:

- WMuCpp repository: <https://github.com/wimalopaan/wmucpp>
- STM32 MultiSwitch application: <https://github.com/wimalopaan/wmucpp/tree/master/boards/rcmultiswitchG030>
- Original entry point: <https://github.com/wimalopaan/wmucpp/blob/master/boards/rcmultiswitchG030/msw30.cc>
- CRSF parameter menu: <https://github.com/wimalopaan/wmucpp/blob/master/boards/rcmultiswitchG030/crsf_cb.h>
- Switch-command handling: <https://github.com/wimalopaan/wmucpp/blob/master/boards/rcmultiswitchG030/switch_cb.h>
- Persistent settings: <https://github.com/wimalopaan/wmucpp/blob/master/boards/rcmultiswitchG030/eeprom.h>
- CRSF protocol/decoder: <https://github.com/wimalopaan/wmucpp/blob/master/include_stm32/rc/crsf_2.h>
- Output/blinking/Morse implementation: <https://github.com/wimalopaan/wmucpp/blob/master/include_stm32/blinker.h>

**Distributed code license: GNU GPL v3 or later (`GPL-3.0-or-later`); see the project’s `LICENSE` file.** Wilhelm Meier’s copyright notices and attribution must remain in derived source files. The ESP32-S3 modifications are distributed under the same terms. The `elrsV3.lua` script and widget are separate components: this document does not assign either a license that has not been verified.

### Credits

- **Wilhelm Meier** — original WMuCpp / RC MultiSwitch-E author; source design, switch protocol and parameter model.
- **Pierrot** — ESP32-S3 port initiative, RadioMaster ER8/EdgeTX hardware wiring and testing, validation of outputs, Intervall, PWM and Morse, requirements and feedback.
- **ChatGPT (OpenAI)** — assistance with Arduino/ESP32-S3 adaptation, CRSF diagnostics and documentation, with hardware validation by Pierrot.

This port is **not an official Wilhelm Meier release**.

## 2. Hardware and wiring

The reference setup uses a **RadioMaster ER8 with ELRS/CRSF**, an **ESP32-S3 Super Mini** and LEDs or suitable output drivers.

| Signal | ESP32-S3 | Connect to |
|---|---|---|
| CRSF RX | **GPIO12** | ER8 receiver TX |
| CRSF TX | **GPIO13** | ER8 receiver RX |
| Ground | **GND** | ER8 receiver GND |
| OUT0 to OUT7 | **GPIO4 to GPIO11** | Eight logical outputs; OUT0 = LED 1, OUT7 = LED 8 |
| RGB status LED | **GPIO48** | Onboard WS2812, depending on Super Mini board variant |
| Console | USB / `Serial` | **115200 baud** |

CRSF uses **420000 baud, 8N1**, over **two separate data wires** plus **common ground**. **GPIO12 is an RX signal input, NOT ground.** The current port does not support single-wire/half-duplex CRSF or automatic baud detection. Use 3.3 V logic levels.

Do not drive power loads directly from ESP32 GPIO: use a series resistor for an LED and suitable MOSFET/driver circuitry for floodlights, motors and relays.

### Output electrical polarity

Set in `devices_3.h`:

```cpp
#define MSW_OUTPUT_ACTIVE_LOW 1
```

| Value | ON level | OFF level |
|---|---|---|
| `1` (port default) | LOW, about 0 V | HIGH, about 3.3 V |
| `0` | HIGH, about 3.3 V | LOW, about 0 V |

Polarity applies to all eight GPIOs, including PWM. **In v0.2h2 this is a global compile-time option: `elrsV3.lua` cannot configure it, nor can individual outputs have different polarities.** It does not change logical ON/OFF commands or the meaning of `outs=XX`.

**ULN2803 warning:** a HIGH input turns its output transistor on and pulls the connected load toward GND. Choose polarity for the actual circuit, not merely the LED type.

## 3. How the port works

1. The ER8 sends CRSF frames to ESP32-S3 GPIO12.
2. The decoder checks CRSF frames (CRC8 polynomial `0xD5`), receives channels and MultiSwitch commands, and checks the **logical switch address**.
3. MultiSwitch **`Set`, `Set4`, `Set4M`** control logical ON/OFF; **`Prop`** supplies the proportional PWM duty.
4. The ESP32-S3 replies to CRSF device-discovery and parameter requests on GPIO13. **`elrsV3.lua` displays the parameters advertised by the firmware**, so the Lua file itself does not need modifications for these new menus.
5. The **`lvglMultiSw` widget** sends CRSF MultiSwitch commands; the Lua parameters configure how outputs respond. For example, a blinking output can also have PWM dimming.
6. Persistent settings use ESP32 **NVS**, namespace `msw-s3`, saved after approximately **3 seconds without further changes**. Wait a few seconds after editing before switching the board off.

The **logical switch address** (`Switch Addr`, `2` on the test setup) differs from the internal **CRSF device address** (`0xC8` in this release). This port has one logical switch address.

### Port development history

| Version | Main change |
|---|---|
| `v0.2` through `v0.2c` | CRSF device/menu dialogue, address, failsafe, ON/OFF outputs, compile-time polarity. |
| `v0.2d` | Faster `HardwareSerial` receive path, CRC lookup, batch processing and larger RX buffer. |
| `v0.2d1` | Optional `C=1 / C? / C=0` raw RX observer without modifying the working decoder. |
| `v0.2f` | **Operate** and per-output **Intervall** blinking. |
| `v0.2g` | Eight hardware **LEDC PWM** channels and **Prop** support. |
| `v0.2h` | **Morse** with one shared text and five timing values. |
| `v0.2h1` | **Repeat** and **Repeat pause**, ESP32-S3-specific extensions. |
| **`v0.2h2`** | Read-only Morse timing help folder in the radio menu. |

The Arduino entry point (`.ino` → `msw30.cpp`) keeps recognizable upstream naming, but GPIO, UART, RGB LED, NVS and PWM backends were adapted to ESP32-S3. Earlier experiments, notably the native-UART `v0.2e` branch, are **not the current baseline**.

## 4. Complete guide to the `elrsV3.lua` options

The menu depends on what the loaded firmware advertises. This guide describes **v0.2h2**, not every feature available in the original STM32 implementation. Original English field labels are preserved to match the radio display.

### Device information and `Global`

| Field | Meaning |
|---|---|
| `Version(HW/SW)` | Internal hardware variant/software information for the ESP32-S3 port, not Wilhelm’s STM32 PCB revision. |
| `Global → Switch Addr` | Logical address accepted by MultiSwitch commands. Match the widget address; saved to NVS. |

**Not currently offered by the Lua menu:** polarity inversion, pin selection, writable CRSF device address, PWM frequency, factory reset, or advanced telemetry settings.

### `Failsafe`

Failsafe is applied when the CRSF link **transitions from connected to disconnected** (no fresh valid channel frames for about 500 ms). It does not suppress normal widget commands while the link is active.

| Field | Meaning |
|---|---|
| `Mode → Hold` | Retain the last requested output states/values during link loss. |
| `Mode → All-Off` | Turn all outputs OFF; PWM outputs go to 0%. |
| `Mode → Set` | Apply individually configured `Set Output 0…7` states. |
| `Set Output 0…7 → Off/On` | Desired output state **only when `Mode=Set` and failsafe occurs**; not a normal manual ON/OFF button. |

**`Set` with all eight values `Off` is equivalent to `All-Off` upon link loss.** Under `Hold`, ongoing blinking or repeating Morse can continue as the output’s ON request is held. In **`PWM Mode=Remote`**, `Set=On` means 100% and `Set=Off` means 0%; the failsafe duty remains until a new `Prop` command arrives. Test motor loads safely without a propeller.

### `Operate`

| Field | Meaning |
|---|---|
| `Output 0…7 → Off/On` | Immediately control any output **from the Lua script** without the widget. It is a temporary command, not a saved active state restored after reboot. |

The widget and `Operate` both issue logical commands. The selected output’s Intervall, PWM and Morse configuration applies. In `PWM Mode=Remote`, duty is governed by `Prop`, not by the ON/OFF button.

### `Output 0` … `Output 7` — individual configuration

Each folder maps to a GPIO: `Output 0` = GPIO4, `Output 7` = GPIO11. All eight share the same engine, while Intervall and PWM settings are independent per output and stored in NVS.

| Field | Range | Effect |
|---|---|---|
| `Intervall Mode` | `Off / On / Morse` | Steady output / grouped flashes / send Morse text. |
| `Intervall(on)` | 1–255, **× 50 ms** | Flash ON duration in `On` mode. |
| `Intervall(off)` | 1–255, **× 50 ms** | Pause between flash groups in `On` mode. |
| `Intervall(count)` | 1–4 | Flashes per group. The space between flashes in one group also uses `Intervall(on)` in this port. |
| `PWM Mode` | `Off / On / Remote / Global/Indiv` | See below. |
| `PWM Duty` | 1–99% | Stored intensity; default 50%. |
| `PWM Expo` | 0–100 | Stored **but currently has no effect**; upstream `expo()` is also empty. |

**`Intervall(on/off/count)` does not control Morse timing.** Use the `Morse` folder for Morse timing.

#### `PWM Mode` in detail

| Mode | Actual v0.2h2 behavior |
|---|---|
| `Off` | Ordinary digital ON/OFF; full output when ON. |
| `On` | Configured PWM intensity, **gated by the widget/Operate**; combines with Intervall and Morse. |
| `Remote` | CRSF **`Prop` (0–100%)** directly sets duty; the ON/OFF button is not the duty control. Starts at 0% until the first `Prop`. |
| `Global/Indiv` | Like `On`, with an internal global duty multiplier. **Lua/virtual global-dimming control is NOT yet implemented; global factor defaults to 100%.** |

PWM uses **hardware LEDC at 1 kHz, 8-bit resolution** on all eight GPIOs, not delay-based PWM or the CRSF UART timer. With active-LOW polarity: 0% = static HIGH/OFF; 100% = static LOW/ON. Exact 0% and 100% are static electrical levels. `Prop` values change runtime RAM without overwriting saved `PWM Duty`.

#### Combined examples

| PWM Mode | Intervall Mode | Effect while output is requested ON |
|---|---|---|
| `Off` | `Off` | Full-power steady output. |
| `Off` | `On` | Full-power flashes. |
| `On` | `Off` | Dimmed steady light. |
| `On` | `On` | Dimmed flashes. |
| `On` | `Morse` | Dimmed Morse flashes. |

These are the **ON/OFF-gated PWM modes**. `Remote` follows `Prop` independently of Intervall/Morse commands.

### `Morse` — shared settings for all eight outputs

An output transmits `Text1` when **`Intervall Mode=Morse`** and receives an ON command from the widget or `Operate`. `Text1`, timings and `Repeat` are **shared across all eight outputs**; each output maintains its own playback state.

| Field | Meaning |
|---|---|
| `Text1` | One shared Morse message, **15 characters maximum**, default `SOS`. Letters, digits, spaces and `. , : ; ? ! - = +` are supported; other symbols are rejected or not encoded. |
| `Dit duration` | Lit duration of a **dot** (`.`), in 100 ms units. |
| `Dah duration` | Lit duration of a **dash** (`-`), in 100 ms units. |
| `Intra S. Gap dur.` | Gap **between marks in the same letter**. |
| `Inter S. Gap dur.` | **Additional time on top of Intra** for a gap between letters. |
| `Inter W. Gap dur.` | **Additional time on top of Intra** for a gap between words (a space in `Text1`). |
| `Repeat → Off/On` | **Off:** one message per OFF→ON command (upstream behavior). **On:** repeat while output ON is requested. This ESP32-S3 extension is not part of Wilhelm’s original menu. |
| `Repeat pause` | Pause **between complete messages**, 1–100 × 100 ms (0.1–10 s); default `9` = 0.9 s. Used when `Repeat=On`. ESP32-S3 extension. |
| `Aide durees` | **Read-only** timing explanations, without changing settings. ESP32-S3 v0.2h2 extension. |

The five Morse timing values are **1–10**, in **100 ms steps**. In **this port**, Intra and Inter must be read carefully:

| Gap | Effective calculation |
|---|---|
| Between dots/dashes in one letter | `Intra × 100 ms` |
| Between letters | `(Intra + Inter S.) × 100 ms` |
| Between words | `(Intra + Inter W.) × 100 ms` |
| Between repeated messages | `Repeat pause × 100 ms` |

**Example of standard Morse timing ratios:** `Dit=1`, `Dah=3`, `Intra=1`, `Inter S.=2`, `Inter W.=6` yields a 100 ms dot, 300 ms dash, 100 ms intra-letter gap, 300 ms between letters and 700 ms between words. `Repeat pause=9` adds 900 ms before the next complete message. These are **example settings**: the firmware does not silently change your stored values.

With `Repeat=Off`, playback ends after one message; switch OFF then ON to play it again. With `Repeat=On`, it repeats until OFF. OFF stops playback immediately, even during a mark or repeat pause. PWM adjusts **brightness**, not Morse timing.

## 5. RGB status LED on GPIO48

| Appearance | Meaning in this firmware |
|---|---|
| **Steady green** | Fresh valid CRSF channel frames are arriving. |
| **Brief blue flash** | A MultiSwitch command has just been processed (about 120 ms), then it returns to green. This is why the widget may periodically flash the status LED even with a healthy link. |
| **Red** | No recent CRSF frames / no usable link for about 1 s or more. |
| **Alternating orange/off** | Recent CRSF frames exist, but no fresh valid channel frame for at least 500 ms. |

Blue is intentionally a visible widget activity indicator; **it does not by itself signal an RX error**.

## 6. USB diagnostic console

Use **115200 baud** and press Enter (`CR` or `LF`):

| Command | Action |
|---|---|
| `D=0` | Silent periodic diagnostics (startup default). |
| `D=1` | Once-a-second `[MSW]`, `[LINK]`, and observer counts if enabled. |
| `D=2` | More detailed UART, menu, save and CRC debugging; use for testing. |
| `D?` | Show debug level; `D` toggles between 0 and 1. |
| `C=1` | Start/reset the optional raw CRSF observer. |
| `C?` | Show its counts and `BEFORE / BAD / AFTER` sample, if available. |
| `C=0` | Stop the observer. |
| `?` or `help` | Command help. |

`[MSW] outs=XX` shows **logically visible/active outputs**, not individual PWM edges. `crcErr` and `[RX-CHECK]` are diagnostic aids: one bad candidate does not by itself establish a faulty wire. `D=0` and `C=0` are appropriate for normal operation.

## 7. Installation and quick check

1. Back up the last known-good build. Open `RCMultiSwitch_ESP32S3.ino` in **Arduino IDE**, select your ESP32-S3 board and a compatible Arduino-ESP32 core (development used the **3.0.7** branch).
2. Check **actual common GND**, crossed TX/RX wiring on GPIO12/13, and safe GPIO loads.
3. Build/flash, then open the USB console at 115200 baud. `Switch Addr` may be read from older NVS settings.
4. Open `elrsV3.lua` on the radio, select MultiSwitch and check `Global`, `Failsafe`, `Operate`, all eight `Output` folders and `Morse`.
5. On OUT0/GPIO4 try ON/OFF, `Intervall Mode=On`, `PWM Mode=On` at 10/50/90%, then `Intervall Mode=Morse`, `Text1=SOS`, `Repeat=On`. Verify OFF stops each mode.
6. Wait **at least 3–4 seconds after the final edit** before restarting; confirm settings persisted.

The **1 kHz lighting PWM is not a 1–2 ms RC servo signal**: this release does not provide servo outputs.

## 8. Not yet ported / planned

These are deliberately distinguished from **features already working in v0.2h2**:

- **Virtuals:** virtual address and grouped physical outputs.
- **Externally commanded Global Dimming:** the internal `Global/Indiv` multiplier exists, but the virtual/Lua control is not yet ported.
- **Patterns:** timed multi-output sequences and chaining.
- Other conditional upstream features: master/slave commands, Reset menu, sensor/advanced telemetry and STM32-board-specific functions.
- **Polarity setting via `elrsV3.lua`** (global or per-output): **proposed but NOT implemented**.
- **Configurable UART serial output** using OUT0…7: **not implemented**, and distinct from CRSF on GPIO12/13.

Do not present these as existing v0.2h2 features. The agreed approach is to retain the proven CRSF receive path and add and validate one feature at a time on actual hardware.

---

**Credits and attribution:** WMuCpp / RC MultiSwitch-E © Wilhelm Meier; ESP32-S3 adaptation and testing with Pierrot and ChatGPT assistance. **Code license: GPL-3.0-or-later.** The original project and this port must remain clearly distinguished.
