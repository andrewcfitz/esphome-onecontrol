# esphome-onecontrol

Lippert / OneControl IDS-CAN in ESPHome and Home Assistant.

## 1. What this is

A set of ESPHome packages that put a Lippert / OneControl coach's lights, tank
levels, water pump, tank heater and battery voltage into Home Assistant, by
talking to the coach's own CAN bus.

It reads and it writes. This is not a passive listener that mirrors the
OneControl app. Home Assistant turns lights on and off, sets brightness on the
fixtures that can do it, and runs the pump and tank heater. State comes back off
the bus, so a wall switch or the phone app moves Home Assistant too, within about
a second.

What you get, once you have listed your own devices:

- Lights as Home Assistant light entities, dimmable where the fixture really
  dims.
- Tank levels as percentage sensors.
- Water pump and tank heater as switches.
- Battery voltage as a sensor.
- A restart button for the ESP board itself.
- A diagnostic state sensor beside each device, showing what the bus says rather
  than what Home Assistant hopes.

There are no slide, awning or leveler controls. Section 2 explains why.

> **TODO: Home Assistant screenshot to be added.**

You need an ESP32 board with a CAN transceiver, a connector, and about an hour.

### Requirements

ESPHome 2025.3.0 or newer. This package instantiates the same device file
repeatedly with different `vars:`, and per-file `vars:` in a remote package's
`files:` list is what 2025.3.0 added. The Home Assistant ESPHome add-on can lag
well behind that; check its version first if the package will not load.

## 2. Read this before you buy anything

This transmits on the CAN bus that your coach's slides, levelers and brakes are
also sitting on. That is not a reason to avoid it. It is a reason to be
deliberate.

Three things keep it safe:

**It only addresses nodes you name.** Every command is aimed at one node that
you described in your own config, by what that node announces itself as. There
is no broadcast, no scan-and-poke, no "turn everything off". If you do not list
your slide, nothing in here will ever send it a frame.

**It does not hold anything open.** A command opens a session with one node,
sends one command, and closes the session immediately. The OneControl phone app
is never locked out and nothing on the bus is left waiting on this board.

**Nothing is configured by address.** Addresses are assigned by the system at
power-up, and they do not survive a full power loss: on the coach this was
built on, a dead house battery moved 15 of the 17 configured nodes. So you do
not write addresses down. Each device is named by its `FUNCTION_NAME` and
function instance, which the node broadcasts about itself and which do not
change, and the board finds where that node is right now. That is how the
OneControl panel does it too.

Each device file also fixes the `DEVICE_TYPE` it will accept. A relay file only
binds to a latching relay and a light file only to a dimmable light, so no
combination of vars can point one at a motor. If two nodes announce the same
identity at once, the device is refused, not guessed at, and the log says so.

### Why there are no motor controls

In the OneControl app, an awning runs only while your finger is on the button.
Let go and it stops. That is not a UI convention, it is enforced on the wire: the
motor node stops when the controlling session ends, and the session expires after
five seconds of silence.

A Home Assistant button is not a finger. There is no equivalent of letting go. A
tap, a scene, an automation and a dropped WiFi connection all look much the same
from the motor's side, and the deadman that makes the app safe does not survive
the translation into a smart-home toggle.

I run awning controls on my own coach. I know what they take. I have
deliberately not published them.

Nothing here targets a motor. There is one command path, in `core.yaml`, and it
sends either a light command or a relay on or off. The motor device type appears
in exactly one YAML file, `discover.yaml`, in a lookup table whose only job is to
tell you which nodes to leave alone.

**That is not the same as saying this package cannot move anything.** A relay ON
is an empty payload with `1` in the message-data byte of the CAN ID. A momentary
H-bridge motor reads exactly that frame as FORWARD. The motor classes derive from
the relay class and inherit its command convention unchanged, so there is no
discriminator on the wire. The discriminator is in the binding: each device
file checks a node's `DEVICE_TYPE` before it will talk to it at all. In v1 the
only safeguard was a hex string you typed into `dev_addr`, and a power cut was
enough to make that string point somewhere else. Do not add lambdas of your own
that send to a raw address, for the same reason.

If it ever happens, the damage is limited but not by design. The command path
opens a session, sends, and closes, and ending the session is itself a stop on a
momentary H-bridge, so the motor moves for a few tens of milliseconds. Repeated
toggles ratchet it further each time. Treat that as a description of the failure
mode, not as a safety feature. It is neither guaranteed by this code nor
something to lean on.

## 3. Hardware

Any ESP32 with a CAN transceiver will do the job. Two boards are known to work,
and both isolate the CAN side. Isolation matters here: on a vehicle, two grounds
several metres apart are rarely the same ground, and a non-isolated transceiver
bridging that difference is a bad afternoon.

### Waveshare ESP32-S3-RS485-CAN (recommended)

The default pick if all you want is OneControl in Home Assistant.

- Isolated CAN, with an onboard 120 ohm termination jumper, disabled by
  default.
- 16MB flash, 8MB PSRAM.
- 7-36V screw terminal input, so it runs off coach 12V directly with no
  converter.
- WiFi only, no wired Ethernet.
- CAN on GPIO15 (TX) and GPIO16 (RX), confirmed from Waveshare's own Arduino
  demo (`WS_GPIO.h`, `TXD2`/`RXD2`).

Board file: `boards/waveshare-esp32-s3-rs485-can.yaml`.

### Waveshare ESP32-S3-ETH-8DI-8RO-C

The pick if you also want eight relays and eight isolated digital inputs on the
same board, for things that are not on the CAN bus at all.

- Isolated CAN, with the same 120 ohm termination jumper as the RS485-CAN
  board, disabled by default.
- Wired Ethernet (W5500). No WiFi needed.
- 16MB flash.
- Same 7-36V screw terminal input.
- 8 relay outputs, 8 isolated inputs, yours to configure. The board file only
  sets up the processor, the CAN pins and the Ethernet PHY.
- CAN on GPIO17 (TX) and GPIO18 (RX), confirmed from Waveshare's own Arduino
  demo (`WS_GPIO.h`, `TXD1`/`RXD1`).

Board file: `boards/waveshare-esp32-s3-eth-8di-8ro-c.yaml`.

Bigger and dearer. It is also the board this was developed on, so its pinout is
confirmed by the fact that it works.

Either way you also need 12V from a fused feed, and the cable in section 4.

## 4. Wiring

The tap used here is the **spare port on the Level Up leveling control board**.
It gets you CANH and CANL without cutting into a harness.

> **TODO: photo of the leveler port to be added.**

That port was occupied by a terminating plug, which had to come out to make room.
Remember that, because section 4.1 turns on it.

### The cable

Molex Mini-Fit Jr., 2 circuit, 4.2mm pitch, receptacle to receptacle, 2m of
18AWG.

| | |
|---|---|
| Molex part | **245135-0220** |
| Pitch | 4.2mm |
| Circuits | 2 |
| Ends | receptacle to receptacle |
| Length | 2m |
| Wire | 18AWG |

Cut it in half. That gives two 1m pigtails: one to use and one spare, which is
worth having when you are working on your back in a basement compartment.

Two circuits means **CANH and CANL only**. There is no power in this cable. The
board's 12V comes from a separate feed, which is the whole point of the 7-36V
screw terminal on both recommended boards.

Strip the cut end, land the two conductors on the board's CAN screw terminals,
and note which colour went where. If you get the two backwards, see section 4.2.
It is not a disaster.

### 4.1 Termination

A CAN bus wants exactly two 120 ohm terminators, one at each physical end. Not
one. Not three.

**The rule: terminate if and only if your tap replaced a terminator, or your
board sits at a physical end of the bus.**

Two cases:

- **You pulled a terminating plug out to make room.** The bus is now one
  terminator short and your board owes it one. **Enable the onboard 120 ohm.**
- **You T-tapped into the middle of a run and both existing terminators are
  still in place.** The bus already has its two. **Leave yours off.**

Three terminators is not extra margin. It loads the bus past what the
transceivers can drive, and the result is not "slightly worse". It is a bus that
may stop working for everything on it, leveler and slides included. Get this one
right.

**For the documented tap** (leveler spare port, terminating plug removed) the
answer is: **enabled**.

Both recommended boards ship with the 120R jumper off. Nothing about flashing
the board file changes that: it is a physical jumper, and it stays wherever it
was left. If you are doing the documented tap and you do nothing else, you end
up with a bus one terminator short. Move the jumper to 120R.

On the RS485-CAN board and the ETH-8DI-8RO-C that is the 120R jumper. On other
boards it may be a DIP switch or a solder bridge. Check your board's own
documentation.

### 4.2 Polarity

Getting CANH and CANL backwards damages nothing. The transceiver does not care,
the bus does not care, nothing gets hot. What happens is that no frames arrive:
the board sits there receiving nothing at all.

Which makes swapping the two wires a safe and cheap first thing to try when
nothing works. Do that before you start suspecting the software.

## 5. Getting it running

The order matters, because you cannot write your device list until you know what
your own nodes call themselves.

**1. Install ESPHome.** The Home Assistant add-on, the CLI, whichever you
prefer. The ESPHome docs cover this better than I can.

**2. Create `secrets.yaml`** with `wifi_ssid`, `wifi_password` and
`api_encryption_key`.

**3. Flash `examples/minimal.yaml`.** Copy it, point it at the board file that
matches your hardware, flash over USB. It includes `discover.yaml` and no
devices, so it cannot command anything. It only listens.

**4. Read the log.** Run `esphome logs your-node.yaml` and watch for lines tagged
`oc.discover`. Every node broadcasts its own identity, nominally once a second and
in practice sometimes only every 15 or so, and you get one line per node:

```
[oc.discover] node 0x21  LATCHING_RELAY(3)  WATER_PUMP(5)  instance 0  ->  dev_func: 5, dev_inst: 0
[oc.discover] node 0xEC  TANK_SENSOR(10)  FRESH_TANK(67)  instance 0  ->  dev_func: 67, dev_inst: 0
[oc.discover] node 0x95  DIMMABLE_LIGHT(20)  KITCHEN_PENDANTS_LIGHT(34)  instance 0  ->  dev_func: 34, dev_inst: 0
[oc.discover] node 0xBC  DIMMABLE_LIGHT(20)  SOFFIT_LIGHT(366)  instance 0  ->  dev_func: 366, dev_inst: 0
```

The part after the arrow goes in your device's `vars`. The address is there to
help you read the log and nothing else: it will be different after the next
power cut.

A full coach has something like 36 nodes. It reports the motors too, which is
deliberate: it is a complete inventory, and the motors are part of it. A type or function the lookup table has no label for prints as `UNKNOWN`
with its raw number beside it, so you can look it up or report it. A node is
worth having whether or not this project knows its name.

Nodes that broadcast no function name at all are skipped rather than printed as
noise. Those cannot be used in v2: with no function name there is nothing stable
to find them by. If a device you want is one of them, open an issue with its
discovery details rather than reaching for an address.

**If discovery prints a node at `0xA0`, stop and open an issue.** That is the
source address this package transmits as, and it is fixed: `0xA0` is baked into
the frame IDs it sends and into the receive filters that decide which replies
are ours, in both `core.yaml` and `battery.yaml`. There is no substitution for
it, so a consumer pinning a release cannot change it. Sharing it with a real node
means your board is impersonating that node: sessions get attributed to the wrong
device, and lights intermittently do not respond with nothing in the log to say
why. The coach this was built on had `0xA0` free. Yours might not.

**5. Write your device list.** One entry per device in the `files:` list,
picking the right file from section 6. `examples/coach.yaml` shows a complete
one. Function names and instances travel better between coaches than addresses
did, but they are still one coach's, so check every one against your own log.

On boot each device logs the node it found, `[oc] Water Pump is node 0x21`,
usually within 20 seconds. One that never does is named wrongly or is on the
wrong device file. Section 8.

**6. Comment out `discover.yaml` and flash again.** It costs a couple of KB of
flash and some log noise at boot. No harm in leaving it, no point either once
you have your list.

### Pin the version

Both examples use `ref: v2.0`. Keep it that way. Tracking `main` means an OTA
update can change what your lights do at a moment you did not choose, which is a
poor property in a vehicle. Bump the ref deliberately, when you feel like reading
a changelog.

```yaml
packages:
  onecontrol:
    url: https://github.com/andrewcfitz/esphome-onecontrol
    ref: v2.0
    files:
      - boards/waveshare-esp32-s3-rs485-can.yaml
      - core.yaml
```

### Upgrading from v1

v2 drops `dev_addr`. Every device entry needs `dev_func` and `dev_inst` instead,
which discovery now prints ready to paste. Include `discover.yaml` once, read
off each device's pair, and replace the address. Nothing else in your device
list changes, and entity ids and names stay the same, so Home Assistant keeps
its history.

If you added lambdas of your own that send to a fixed address, they have the
same problem v1 did. `id(oc_resolve)(1ull << DEVICE_TYPE, FUNCTION_NAME,
instance)` returns the node's current address, -1 for none, or -2 if it is
ambiguous. Send nothing if it is negative.

### If you add your own `on_boot`

`core.yaml` declares `esphome.on_boot` as a list, not a mapping, specifically so
consumers can too. ESPHome merges packages by walking into mappings and
concatenating their `then:` lists, but it concatenates two lists as separate
automations, ordered by priority. Write yours as a list and you get a second,
independent automation. Write it as a mapping and ESPHome merges it into core's
single automation, tacking your actions on after core's `delay: 10s`.

```yaml
esphome:
  on_boot:
    - priority: -10
      then:
        - logger.log: "our own boot action"
```

## 6. Device reference

`core.yaml` is required. It sets up the CAN bus, the session handshake, the
command path, and a Restart button for the board. Include exactly one file from
`boards/` before it, or set `can_tx_pin` and `can_rx_pin` yourself.

Everything under `devices/` is included once per physical device, with vars. All
of them take:

| Var | Meaning |
|---|---|
| `dev_id` | ESPHome id prefix, e.g. `hutch`. Lowercase, no spaces. |
| `dev_name` | Home Assistant name, e.g. `Hutch`. |
| `dev_func` | The node's `FUNCTION_NAME` number, e.g. `5` for a water pump. |
| `dev_inst` | Its function instance, e.g. `2` for the second awning light. |
| `dev_icon` | An `mdi:` icon. |

Take `dev_func` and `dev_inst` from `discover.yaml`, which prints both. The
`DEVICE_TYPE` is not a var: each file fixes the type it accepts, so a node of
the wrong class is never bound, whatever its function name.

A device that is not bound neither sends nor listens. Its entities stay unknown
and a command logs a warning instead of reaching the bus. Binding takes up to
about 20 seconds after boot, because some nodes announce themselves that rarely.

Discovery does not list nodes that broadcast no function name, and v2 cannot
bind them. See section 5.

### Which light file

Three files can drive a light. Picking the wrong one is the mistake most likely
to cost you an evening.

| Your node reports | What the fixture does | Use |
|---|---|---|
| `DIMMABLE_LIGHT` | dims properly | `devices/dimmable_light.yaml` |
| `DIMMABLE_LIGHT` | strobes when dimmed | `devices/dimmable_light_switched.yaml` |
| `LATCHING_RELAY` | on and off only | `devices/relay_light.yaml` |

The thing that catches people: **`DEVICE_TYPE` describes the driver, not the
fixture.** A node can report `DIMMABLE_LIGHT`, accept a brightness command
without complaint, and then visibly strobe on anything below full, because the
LED module wired to it cannot really dim. Only your coach can tell you which of
yours are which. Turn each one down to 50% and look at it.

This matters for more than the entity type, because **device class governs the
command form on the wire**:

- A dimmable node takes an **8-byte payload**, mode in byte 0 and brightness in
  byte 1.
- A latching relay takes an **empty payload**, with the command in the
  message-data byte of the CAN ID.

Each silently discards the other. No error, no NAK, no log line. So using the
wrong file does not degrade the light, it stops it responding at all.

Which is why the strobing case gets its own file instead of being sent to
`relay_light.yaml`. `dimmable_light_switched.yaml` gives you an on/off entity
with no slider, while still speaking the dimmable command form its node expects.

### `devices/dimmable_light.yaml`

A `DIMMABLE_LIGHT` node whose fixture really dims. Binds only to
`DIMMABLE_LIGHT`.

Vars: `dev_id`, `dev_name`, `dev_func`, `dev_inst`, `dev_icon`.

Entities:

- **`${dev_name}`**, a monochromatic light with a brightness slider.
- **`${dev_name}` State**, a diagnostic binary sensor, device class `light`.
  Ground truth from the bus. It reads unknown until the node speaks, which is
  honest.
- **`${dev_name}` Level**, a diagnostic percentage sensor: byte 1 of the node's
  own report, 0-100. It is also the feedback path that keeps the slider in step
  with reality, and the only direct evidence of whether a fixture really dims or
  merely reports 100 whenever it is on.

Transitions are set to 0s and gamma correction to 1.0, both deliberately. The
comments in the file give the reasoning.

### `devices/dimmable_light_switched.yaml`

A `DIMMABLE_LIGHT` node whose fixture does not dim. Binds only to
`DIMMABLE_LIGHT`.

Vars: `dev_id`, `dev_name`, `dev_func`, `dev_inst`, `dev_icon`.

Entities:

- **`${dev_name}`**, a binary light. On and off, no slider.
- **`${dev_name}` State**, diagnostic binary sensor, as above.
- **`${dev_name}` Level**, diagnostic percentage sensor. Kept even though there
  is no slider to push it into, because it is how you found out the fixture does
  not dim and it is how the next person will.

Commands go out in the dimmable 8-byte form, always at level 100.

### `devices/relay_light.yaml`

A light on a `LATCHING_RELAY` node. Binds only to `LATCHING_RELAY` or
`LATCHING_RELAY_TYPE_2`.

Vars: `dev_id`, `dev_name`, `dev_func`, `dev_inst`, `dev_icon`.

Entities:

- **`${dev_name}`**, a binary light.
- **`${dev_name}` State**, diagnostic binary sensor, device class `light`.

Not optimistic: the light entity follows what the node reports, so a rejected
command springs the entity back within a second instead of lying to you. It also
watches commands other clients send, so the phone app and the wall keypads move
Home Assistant within milliseconds rather than at the next 1 Hz report.

### `devices/relay_switch.yaml`

A `LATCHING_RELAY` node that is not a light: water pump, tank heater, and
anything similar. Binds only to `LATCHING_RELAY` or `LATCHING_RELAY_TYPE_2`.

Vars: `dev_id`, `dev_name`, `dev_func`, `dev_inst`, `dev_icon`, plus:

| Var | Meaning |
|---|---|
| `dev_class` | Home Assistant device class for the state sensor. `running` for a pump, `heat` for a heater. Defaults to `running`. |

Entities:

- **`${dev_name}`**, a template switch.
- **`${dev_name}` State**, diagnostic binary sensor with your `dev_class`.

The switch reads unknown, not off, until its node has said something. An
unreported pump is not a pump that is off, it is a pump you have not heard from.

### `devices/tank.yaml`

One tank level sensor. Binds only to `TANK_SENSOR`.

Vars: `dev_id`, `dev_name`, `dev_func`, `dev_inst`, `dev_icon`.

Entities:

- **`${dev_name}`**, a percentage sensor.

Tank nodes just broadcast, 1 Hz idle and faster on change. No session, no
polling, nothing is transmitted to them at all.

Expect coarse numbers. These are three-segment probes and report in 33% steps.
`33` means one segment wet, not precisely one third of a tank. Gallons are not
available: `IDS-CAN.md` records why that was attempted twice and abandoned.

### `battery.yaml`

Battery voltage. Optional, include once, nothing else depends on it. No vars.

Entities:

- **Battery Voltage**, a voltage sensor in V.

Voltage is not broadcast by anything, so this asks for it every 30 seconds. It
asks whichever chassis info, generator and leveler modules are on the bus,
found by `DEVICE_TYPE` with nothing to configure, and prefers them in that
order, so the reading sticks to one source instead of alternating between them.
With none of the three present the sensor stays unknown and nothing else
breaks. Implausible readings are logged
and discarded rather than published, because a believable wrong number is worse
than a gap.

### `discover.yaml`

Include it to find your devices' `dev_func` and `dev_inst`, remove it when you
are done. It creates no
entities, only `oc.discover` log lines. See section 5.

## 7. How the protocol was worked out

IDS-CAN is Lippert's own protocol, running at 250 kbit/s. It is not RV-C, not
J1939 and not CANopen, though it borrows the 29-bit source/destination habit from
one of those. There is no published specification.

Almost none of it was deduced from the wire alone. The bus gave up the shape of
things: which IDs move, what is periodic, what changes when you operate exactly
one device while capturing. The meaning came from a decompiled copy of Lippert's
own firmware, and specifically from the device-side code that *receives and
validates* a message, which is what matters when you are the one transmitting.
An earlier version read only the app's sending code, and as a result every relay
on the coach silently ignored it: no error, no NAK, no log line, just lights that
did not come on.

The coach settled the arguments. Whether a command actually works, and whether a
fixture can do what its driver claims, are not questions a capture or a
decompiler can answer. Every conclusion that skipped an independent check turned
out wrong, and each of those is written up rather than quietly deleted.

**[`IDS-CAN.md`](IDS-CAN.md)** is the full account: frame layouts, the session
handshake and its modified-TEA seed/key exchange, the device inventory format,
the two opposite command conventions, the wrong turns with firmware citations,
and an explicit list of what is still unverified. Read it if you want to extend
this, or if you enjoy watching somebody argue with a decompiler.

## 8. Troubleshooting

### No frames at all

Nothing in the log, no `oc.discover` lines, no entity ever leaves unknown. In
rough order of likelihood:

1. **Polarity.** CANH and CANL swapped. Harmless, and the most common cause.
   Swap the two wires and try again.
2. **Wrong pins for your board.** `can_tx_pin` and `can_rx_pin` have to match
   your hardware. Using the wrong board file is the usual way to get this
   wrong.
3. **Termination left at its default.** Both recommended boards ship with the
   120R jumper disabled. Section 4.1. Too few and too many both stop the bus.
4. **Bit rate.** IDS-CAN is 250 kbit/s. `core.yaml` sets it, so this only bites
   if you overrode it.
5. **Not actually connected.** Check the connector is seated and that the screw
   terminals have copper in them rather than insulation.

### Entities stay unknown

Frames are arriving, since you saw `oc.discover` lines, but one device never
reports.

- **Never bound.** Look for `[oc] <name> is node 0x..` in the log. If it never
  appears, `dev_func` or `dev_inst` does not match anything on the bus, or the
  node is a different `DEVICE_TYPE` from the one the file accepts. Check both
  against discovery.
- **Refused as ambiguous.** `[oc] <name>: more than one node announces...` means
  two nodes claim the same function and instance. If this board stayed powered
  while the coach reshuffled, it clears within a minute as the old address ages
  out. If it persists, two of your nodes really do share an identity; open an
  issue.
- **The node is not present.** Not every coach has every device. A tank heater
  you do not have will never report.
- **It has not spoken yet.** Status arrives once a second, but a node can take
  up to about 20 seconds to announce who it is, and nothing is believed before
  that. Longer than that means one of the above.

"Unknown" here is deliberate. Nothing guesses a default state, because a pump
reported as off before anything has said so is a guess dressed up as a reading.

### A light flashes instead of fading

The fixture cannot dim, whatever its `DEVICE_TYPE` says. Move it from
`dimmable_light.yaml` to `dimmable_light_switched.yaml` and it becomes a clean
on/off entity that still speaks the command form its node expects. See section 6.

### A light never binds, though discovery lists it

Wrong device file. Each file binds only to the `DEVICE_TYPE` whose command form
it speaks, so a relay file pointed at a dimmable node, or the reverse, never
binds at all. In v1 this showed up as a light that reported state and ignored
commands; now it fails earlier and says less. Check discovery's output for that
node: `LATCHING_RELAY` means `relay_light.yaml`, `DIMMABLE_LIGHT` means one of
the two dimmable files. Section 6 has the table.

### Commands ignored

- **It is not bound.** A command to an unbound device logs `no node bound,
  command not sent`. See "Entities stay unknown" above.
- **The node is fault-latched.** A relay that has latched out on a fault stops
  accepting commands until the latch is cleared. The OneControl app or a power
  cycle will do it.
- **Session trouble.** Rare, but if this board ever gets confused about a
  session, the Restart button drops every session it holds and the coach carries
  on without it. Nothing here is remembered across a reboot by design, so a
  restart costs only the few seconds it takes the bus to report everything again.

### Everything on the coach misbehaves

Disconnect this board and see whether the problem goes away. If it does, suspect
termination first (section 4.1). Three terminators on a bus is not a subtle
fault, and it affects every node, not just yours.

## 9. Unofficial

This project is not affiliated with, endorsed by, or supported by Lippert
Components, OneControl, Grand Design (the coach it was developed on), Brinkley
RV, or any other RV manufacturer using this hardware. All product names and
trademarks belong to their respective owners.

It was worked out by reverse-engineering, it talks to systems in a vehicle, and
it comes with no warranty of any kind. Use it at your own risk.
