# Lotus Lantern BLE LED Strip Support Research

Research date: 2026-09-09

## 1. Decision summary

Brobot should support the extra strip by keeping its existing Bluetooth
controller and making the Pico W a Bluetooth Low Energy (BLE) Central. The
strip is a decorative, non-safety output. It must never replace the onboard
RGB status LED or delay motor stopping, battery protection, Wi-Fi handling, or
OTA safety.

The recommended first implementation is deliberately small:

1. Identify the exact controller profile on the physical strip.
2. Connect and implement power, solid RGB color, and brightness.
3. Add reconnect and commanded-state handling.
4. Add white/color-temperature and built-in effects only when the identified
   profile proves that the hardware supports them.
5. Add robot interactions in a later layer that calls this driver. Do not put
   interaction or emotion rules inside the BLE transport.

Direct Pico W control is technically possible, but the current Arduino-Pico
high-level BLE client is not automatically suitable for this controller. Two
compatibility checks are required before implementation:

- Many ELK-family controllers expose a write-*without-response*
  characteristic. Arduino-Pico 6.0.0 reports that capability, but its public
  `setValue()` path checks normal write support and performs a write with a
  response. A controller that exposes only write without response will need a
  small, pinned core extension or a raw BTstack GATT client.
- Arduino-Pico 6.0.0 still marks random BLE addresses as not handled. The
  address type of the actual controller must therefore be recorded during the
  hardware-identification spike.

These are implementation risks, not proof that the strip is incompatible.

## 2. What was researched

The main reference is
[`dave-code-ruiz/elkbledom`](https://github.com/dave-code-ruiz/elkbledom), a
Home Assistant integration for inexpensive BLE controllers used by Lotus
Lantern, Lotus Lamp X, duoCo Strip, and similar apps. Findings in this document
are pinned to upstream commit
[`f41b8a2`](https://github.com/dave-code-ruiz/elkbledom/tree/f41b8a27838f5b76f2b8b2c628b61cd700fe5fc3).

That repository is useful as a protocol and device-profile reference. Its
Python/Home Assistant connection code should not be copied as a firmware
architecture: Brobot runs on a Pico W, has a real-time safety loop, and uses
the Arduino-Pico core.

No physical strip was available during this research. The packet values below
are source-derived candidates and have **not** been validated on Brobot's
controller.

## 3. Device-family findings

“Works with Lotus Lantern” identifies an ecosystem, not one protocol variant.
The reference repository contains profiles for names including:

- `ELK-BLEDOM`, `ELK-BLEDOB`, `ELK-BLEDDM`, `ELK-BLE`, and `ELK-BTC`
- `MELK` and several `MELK-O...` variants
- `LEDBLE`, `LED-`, `JACKYLED`, and `XROCKER`

The same advertised name can still require different commands. In particular,
the reference has more than one `ELK-BLEDOM` profile and can refine selection
using the GATT characteristic handle. Brobot must therefore use a configured
and recorded hardware profile, not guess forever from the name alone.

### Common GATT layouts

The common ELK layout observed by the reference project is:

| Purpose | UUID | Typical properties |
|---|---|---|
| Service | `0000fff0-0000-1000-8000-00805f9b34fb` | Primary service |
| Command write | `0000fff3-0000-1000-8000-00805f9b34fb` | Often write without response |
| Status/echo | `0000fff4-0000-1000-8000-00805f9b34fb` | Notify on some models |

Some `LEDBLE`, `LED-`, and `XROCKER` profiles instead use `FFE1` for writes
and `FFE2` or `FFE1` for reads. UUID matching is more reliable than a
hard-coded attribute handle; the handle should only help select a profile
after service discovery.

The reference integration sends command bytes without a GATT response. It
also notes that only one client can normally be connected, so the Lotus
Lantern phone app must be fully disconnected while Brobot owns the strip.

### Common frame shape

Most basic ELK-family commands are nine-byte frames:

```text
7E  VV  CC  payload (5 bytes)  EF
|   |   |                       |
|   |   command                 frame end
|   profile/variant
frame start
```

`VV` is not a length. It varies by controller and sometimes by command. `CC`
commonly means:

| Command | Meaning |
|---:|---|
| `0x01` | White or brightness |
| `0x02` | Effect speed |
| `0x03` | Built-in effect |
| `0x04` | Power |
| `0x05` | RGB or color temperature |

The common frames have fixed start/end markers and no checksum field is
apparent in the profile data. Other families can change the frame start,
ending byte, length, byte order, and payload semantics.

### Candidate generic `ELK-BLEDOM` commands

These are the generic, no-handle-override profile from the upstream
[`models.json`](https://github.com/dave-code-ruiz/elkbledom/blob/f41b8a27838f5b76f2b8b2c628b61cd700fe5fc3/custom_components/elkbledom/models.json).
They are starting points for a bench test, not universal Lotus Lantern
commands.

| Operation | Candidate bytes |
|---|---|
| Power on | `7E 00 04 F0 00 01 FF 00 EF` |
| Power off | `7E 00 04 00 00 00 FF 00 EF` |
| Solid RGB | `7E 00 05 03 RR GG BB 00 EF` |
| Brightness | `7E 00 01 II FF 00 FF 00 EF` |
| Effect speed | `7E 00 02 SS 00 00 00 00 EF` |
| Built-in effect | `7E 00 03 EE 03 00 00 00 EF` |

`RR`, `GG`, and `BB` are byte values from 0 to 255. In this profile `II` is
an integer percentage from 0 to 100; the reference integration converts its
0-to-255 UI brightness to this range. `SS` and `EE` depend on the selected
profile. The reference defines 22 standard effect IDs, but other controller
families use different lists and ranges.

Examples of why profile selection matters:

- `ELK-BLEDDM` changes the power and brightness variant byte to `0x04`.
- `ELK-BLEDOB` uses `0x07` in several commands and needs a 100 ms gap between
  writes in the reference integration.
- `LED-` and `JACKYLED` swap the green and blue placeholders in their color
  command.
- `XROCKER` uses `0x7B`/`0xBF` frame markers instead of `0x7E`/`0xEF`.
- Some `MELK` devices require an initialization/login sequence before normal
  service use.

Do not probe every profile rapidly against an unknown controller. Select a
profile from its name, UUIDs, properties, and handle, then test a bounded set
of obvious commands with the strip visible on a bench.

## 4. Pico W feasibility and constraints

The Pico W hardware supports BLE Central and Peripheral roles. The official
Pico SDK provides BTstack integration, and the Arduino-Pico core used by the
room-controller project now exposes a BLE client API.

For Arduino-Pico 6.0.0:

- Bluetooth must be enabled through the `Tools > IP/Bluetooth Stack` build
  option. The core documentation estimates roughly 80 KB of flash and 20 KB
  of RAM when Bluetooth is enabled.
- The BLE client can scan, connect, discover services/characteristics, write,
  and subscribe to notifications.
- Wi-Fi and Bluetooth share the Pico W's CYW43439 radio path. Concurrent use
  is supported by the platform, but Brobot must measure latency and stability
  with its web server, WebSocket, mDNS, OTA, and future motor-control loop all
  active.

### High-level Arduino-Pico API gap

The commonly reported `FFF3` characteristic has properties `0x06`, meaning
read plus write without response in the reference GATT dump. The reference
integration explicitly writes with `response=False`.

In Arduino-Pico 6.0.0,
[`BLERemoteCharacteristic`](https://github.com/earlephilhower/arduino-pico/blob/6.0.0/libraries/BLE/src/BLERemoteCharacteristic.cpp)
has `canWriteWithoutResponse()`, but `setValue()` checks `canWrite()` and calls
the response-producing BTstack write function. There is no public
`setValueWithoutResponse()` method in that release. Therefore:

1. Inspect the actual write characteristic properties.
2. If normal write is supported and works, the high-level API may be enough.
3. If it is write-without-response only, use a minimal raw BTstack client or a
   narrowly scoped, version-pinned Arduino-Pico extension. Do not silently
   modify a developer's global Arduino core.
4. Prefer contributing a general write-without-response fix upstream if a
   core change is needed.

### Blocking and real-time safety

The Arduino-Pico
[`BLEClientDemo`](https://github.com/earlephilhower/arduino-pico/blob/6.0.0/libraries/BLE/examples/BLEClientDemo/BLEClientDemo.ino)
uses synchronous scans, connections, and characteristic calls. That is fine
for a demo but conflicts with Brobot's rule that a delayed network or accessory
operation must not delay motor stopping.

The production driver should therefore be event-driven on raw BTstack, or it
must strictly confine potentially blocking discovery/reconnection work to a
disarmed state. If the strip link drops while the robot is armed, mark the
strip unavailable; do not spend seconds scanning or connecting in the body
control path. LED failure must never become a robot fault unless a future
product requirement explicitly says so.

### Address-type risk

Arduino-Pico 6.0.0's `BLEAddressType` includes a random-address value but marks
it as not yet handled. Record whether the physical controller advertises a
public or random address. If it is random and the high-level client cannot
reconnect reliably, the raw BTstack path is the appropriate fallback.

## 5. Proposed Brobot driver boundary

Keep protocol transport separate from robot behavior:

```text
firmware/
  body/
    external_led_strip.cpp       desired state and non-blocking lifecycle
    external_led_strip.h
    elkbledom_protocol.cpp       profile data and packet construction
    elkbledom_protocol.h
```

The exact filenames may change when firmware is imported. The boundary should
remain: protocol code turns typed requests into bytes, while the connection
manager owns scanning, GATT discovery, serialized writes, pacing, retry, and
health.

Suggested application-facing operations:

```text
requestPower(bool on)
requestColor(uint8_t red, uint8_t green, uint8_t blue)
requestBrightness(uint8_t level)        // Brobot API: 0..255
requestEffect(effect, speed)            // later/optional
tick(now_ms)                             // bounded, non-blocking work
status()                                 // reachability and commanded state
```

The protocol profile performs brightness conversion. At the Brobot boundary,
brightness zero should mean off; levels 1 to 255 map to the profile's usable
range.

### Lifecycle

Use an explicit state machine such as:

```text
DISABLED -> SCANNING -> CONNECTING -> DISCOVERING -> READY
                ^                              |         |
                +----------- BACKOFF <---------+---------+
```

Requirements:

- Store one configured device identity and one verified protocol profile.
- Serialize writes; never interleave frame bytes or issue overlapping GATT
  operations.
- Start with a 100 ms minimum command interval, coalesce rapid color/brightness
  requests, then reduce the interval only after measurement. This conservative
  default matches the slowest common profile currently identified.
- Bound reconnect attempts and use backoff. Do not reconnect continuously.
- Reapply the latest desired state once after a successful reconnect.
- Provide a maintenance/release operation that disconnects Brobot so Lotus
  Lantern can connect.
- Never expose a network endpoint that accepts arbitrary raw BLE bytes.

### State semantics

Write without response confirms only that the local stack accepted a packet;
it does not prove that the strip changed. Notifications are inconsistent and
the reference integration currently keeps optimistic local state for several
models.

Brobot should distinguish:

- `desired`: the last state requested by Brobot;
- `commanded`: the last state whose write was accepted locally;
- `observed`: state decoded from a valid notification, when supported;
- `reachable`, `last_error`, and `last_write_ms`: transport health.

Dashboard/API wording must not label a commanded-only state as physically
confirmed.

## 6. Basic feature scope

### Version 1 required

- Enable/disable the feature through configuration.
- Discover or select exactly one configured controller.
- Connect, discover the expected characteristic, and reject unexpected GATT
  layouts.
- Power on and off.
- Set solid RGB color.
- Set brightness without cumulative RGB scaling errors.
- Serialize/coalesce writes and recover from power cycling or radio loss.
- Report connection health and commanded state.
- Preserve all Brobot safety and existing room-controller behavior when the
  strip is absent, busy, incompatible, or powered off.

### Deferred until the profile is proven

- Native white and RGBW/color-temperature control.
- Built-in effects and effect speed.
- Microphone/music modes, schedules, and time synchronization.
- Multiple strips, groups, segments, or addressable-pixel control.
- Emotion, movement, voice, AI, or other interaction rules.

The future interaction layer should use semantic requests and a priority
policy. Safety/fault indication remains on reliable local hardware; an
interaction must not make the BLE strip authoritative for robot state.

## 7. Physical hardware checklist

BLE means there is no GPIO data wire between the Pico W and the strip
controller. Keep the controller supplied with its rated voltage and enough
current for the entire strip. Do not power the strip from the Pico W's 3.3 V
rail or a GPIO pin.

Before mounting it on the robot, record:

- Controller label, strip type (RGB/RGBW/addressable), rated voltage, and
  maximum current or included PSU rating.
- Whether it uses its own PSU or shares the robot battery through a suitable
  regulator.
- Peak-current and brownout behavior at full white brightness.
- Radio range with the final mounting position and motor system active.
- A mounting location that keeps metal, wiring bundles, and the strip
  controller away from the Pico W antenna area where practical.

If the strip shares the robot battery, power integrity and motor-noise testing
are required before treating bench results as mounted-hardware proof.

## 8. Required hardware-identification record

Fill this in before writing the production profile:

```text
Product/label:
Lotus Lantern app version:
Advertised BLE name:
BLE address:
Address type (public/random):
Advertisement service UUIDs:
Primary service UUID:
Write characteristic UUID:
Write properties:
Write value handle:
Read/notify characteristic UUID:
Notifications observed (yes/no):
Pairing required (yes/no):
Verified power-on bytes:
Verified power-off bytes:
Verified red/green/blue bytes:
Verified brightness min/mid/max bytes:
Minimum reliable gap between commands:
Behavior after controller power cycle:
```

Use a BLE inspector first. If the reference profile does not work, capture one
isolated action at a time from Lotus Lantern. The upstream
[`sniffing_ble_device.md`](https://github.com/dave-code-ruiz/elkbledom/blob/f41b8a27838f5b76f2b8b2c628b61cd700fe5fc3/sniffing_ble_device.md)
and
[`LED_DISCOVERY_TOOLS.md`](https://github.com/dave-code-ruiz/elkbledom/blob/f41b8a27838f5b76f2b8b2c628b61cd700fe5fc3/LED_DISCOVERY_TOOLS.md)
describe capture and comparison workflows. Do not commit MAC addresses or raw
captures if they are considered private device identifiers.

## 9. Implementation and validation plan

### Step A: isolated protocol spike

1. Power the strip from its original supply on a bench.
2. Fully close Lotus Lantern so it releases the BLE connection.
3. Record the identification fields above.
4. Test only power off/on, then red, green, blue, and three brightness levels.
5. Confirm whether writes require no response and whether notifications are
   useful.
6. Save the exact verified profile in Brobot rather than keeping broad model
   guessing in production.

### Step B: firmware driver

1. Pin an Arduino-Pico version with BLE enabled and record its flash/RAM cost.
2. Implement the smallest working GATT transport for the verified address type
   and write property.
3. Add pure packet-construction tests for all clamped values and exact frames.
4. Add the non-blocking lifecycle, queue coalescing, backoff, and state model.
5. Keep the feature disabled by default until bench validation passes.

### Step C: Brobot regression testing

Test at least:

- Strip absent at boot.
- Strip already connected to Lotus Lantern.
- Controller power cycled while Brobot is disarmed and while armed.
- Repeated red/green/blue and brightness 0, 1, midpoint, and maximum.
- Rapid slider updates; only the latest desired value should win.
- Web dashboard, REST, WebSocket, mDNS, and OTA with BLE connected.
- Wi-Fi loss and BLE loss independently.
- Motor dead-man timeout while BLE traffic is busy.
- IR receive/transmit while BLE is connected.
- Full-white strip load and motors running from the final power system.

The release evidence must keep four results separate: packet unit tests,
firmware compilation, bench BLE behavior, and final mounted robot behavior.

## 10. Acceptance criteria for basic support

Basic support is complete only when:

- The physical controller profile and address type are recorded.
- Power, RGB, and brightness pass repeated cold-start tests on the real strip.
- A disconnected or unavailable strip never stalls the main loop or weakens a
  motor-stop condition.
- Reconnection is bounded and does not cause a command flood.
- Brobot reports commanded versus observed state honestly.
- The phone app can regain control through a documented maintenance/release
  action.
- Existing smart-home, robot-control, and OTA regression checks still pass.

## 11. Licensing and provenance

`elkbledom` is MIT-licensed. Protocol facts and independently written packet
construction can be used with attribution. If source code is copied or
substantially adapted, preserve its MIT copyright and permission notice as
required by the upstream
[`LICENSE`](https://github.com/dave-code-ruiz/elkbledom/blob/f41b8a27838f5b76f2b8b2c628b61cd700fe5fc3/LICENSE).

The preferred Brobot approach is to implement a small device-specific driver
from the verified wire behavior and cite the reference, rather than importing
the Home Assistant integration.

## 12. Sources

- [`elkbledom` README and compatibility notes](https://github.com/dave-code-ruiz/elkbledom/tree/f41b8a27838f5b76f2b8b2c628b61cd700fe5fc3)
- [`models.json` protocol profiles](https://github.com/dave-code-ruiz/elkbledom/blob/f41b8a27838f5b76f2b8b2c628b61cd700fe5fc3/custom_components/elkbledom/models.json)
- [`definitions.json` effect IDs](https://github.com/dave-code-ruiz/elkbledom/blob/f41b8a27838f5b76f2b8b2c628b61cd700fe5fc3/custom_components/elkbledom/definitions.json)
- [`elkbledom.py` connection, pacing, and state behavior](https://github.com/dave-code-ruiz/elkbledom/blob/f41b8a27838f5b76f2b8b2c628b61cd700fe5fc3/custom_components/elkbledom/elkbledom.py)
- [Arduino-Pico 6.0.0 BLE client documentation](https://arduino-pico.readthedocs.io/en/stable/ble.html)
- [Arduino-Pico Bluetooth setup and resource notes](https://arduino-pico.readthedocs.io/en/stable/bluetooth.html)
- [Raspberry Pi Pico W wireless capabilities](https://www.raspberrypi.com/documentation/microcontrollers/pico-series.html)
- [Raspberry Pi Pico SDK networking and BTstack libraries](https://www.raspberrypi.com/documentation/pico-sdk/networking.html)
