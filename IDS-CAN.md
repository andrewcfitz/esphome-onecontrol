# Reverse-engineering the OneControl bus

How the CAN bus behind this repository was worked out: what the sources were,
how each claim was checked, and which conclusions are still guesses.

The short version: almost nothing here was deduced from the wire alone. The bus
told us the *shape* of things, a decompiled copy of Lippert's own firmware told
us the *meaning*, and the coach itself settled the arguments. Every step where
one of those was skipped produced a wrong answer, and all of those are recorded
below.

## The system

A Grand Design coach on a Lippert / OneControl system. One CAN bus, 250 kbit/s,
about 36 devices, reached with a Waveshare ESP32-S3-ETH-8DI-8RO-C running
ESPHome. Goal: lights, tanks and utilities in Home Assistant.

The protocol is **IDS-CAN**, Lippert's own. It is not RV-C, not CANopen and not
J1939, though it borrows the 29-bit source/destination habit from the latter.
The early guesses that it was one of those are covered under "wrong turns".

## Sources

Four kinds, in descending order of how much they were trusted.

### 1. The coach itself

The final authority, and the only one that settles what is physically wired.
Two things were only ever resolved this way:

- **Whether a command works.** The relay bug below produced no error, no NAK
  and no log line. Only the lights not turning on revealed it.
- **Whether a fixture can do what its driver claims.** Node `0xBC` is a
  `DIMMABLE_LIGHT` and accepts brightness without complaint. The fixture strobes
  on anything below full. The bus cannot tell you this and neither can the
  firmware.

### 2. Captures from this bus

ESPHome logging every frame, taken while deliberately operating one thing at a
time. Two captures did most of the work:

- An idle capture, ~5000 frames, which yielded the device inventory.
- A capture taken while toggling one light repeatedly from the OneControl phone
  app, which yielded the command and session frames.

Operating *one* thing while capturing is what makes this tractable: the light
being toggled was the only node on the bus that changed.

### 3. Lippert's decompiled firmware

The single richest source, and where every constant, payload layout and command
format in this repository ultimately comes from.

- `IDS.Core.IDS_CAN.dll` and `OneControlDevices165.dll`, decompiled with
  `ilspycmd`.
- Obtained via **[manos/OneControl-RV-C-Protocol](https://github.com/manos/OneControl-RV-C-Protocol)**,
  which carries decompiled C# under `tools/decompile/decompiled/`.

All `file:line` citations below refer to that tree. The assemblies are
abbreviated:

| Short | Path |
|---|---|
| `a0078` | `IDS_Core_CAN/assembly_0078/assembly_0078_Sharp.decompiled.cs` |
| `a0079` | `IDS_Core_CAN/assembly_0079/assembly_0079_Sharp.decompiled.cs` |
| `a0080` | `IDS_Core_CAN/assembly_0080/assembly_0080_Sharp.decompiled.cs` |
| `d0165` | `OneControlDevices165/assembly_0165_P.decompiled.cs` |
| `cobs`  | `IDS_COBS/ids.portable.common/` |

The most useful discovery in that tree was the **device-side simulator
interfaces** (`*_SIM_INTERFACE` in `a0079`). These implement how a device
*receives* and validates a message, which is what actually matters when you are
the one transmitting. Reading only the app-side sending code is what caused the
relay bug.

### 4. Another owner's capture

**[D-Jeffrey/UnityX-canbus](https://github.com/D-Jeffrey/UnityX-canbus)**, a
capture and partial decode from a different Grand Design coach.

Valuable precisely *because* it is a different coach: it provides frames this
bus never produced, which makes it an independent check rather than a
restatement. The PID read format and the battery-voltage scaling were both
confirmed against it before either was implemented here.

Its own parser was **not** trusted. Its 5+3 bit split of the device-type field
was tested against its own data and disproved (14 of 15 devices had exactly one
"type"), so only its raw frames were used.

## Method

The loop that worked:

1. **Capture while operating one thing.** Isolates the signal.
2. **Find the shape.** Which IDs move, what changes, what is periodic.
3. **Find the meaning in the firmware.** Never infer a constant that can be
   looked up.
4. **Verify against something independent** before writing code: a second
   capture, another coach's traffic, or arithmetic that must reproduce a known
   frame exactly.
5. **Implement, then test on the coach.** Steps 1-4 cannot tell you what is
   physically wired.

Step 4 is the one that was skipped twice, and both times it cost a release.

### Verification techniques that earned their keep

- **Reproducing captured frames from a formula.** The ID construction was
  accepted only once it regenerated all five captured app frames byte for byte.
- **Round-tripping a decode against a known-plausible value.** The voltage
  scaling was confirmed because `0x0D7FF3 / 65536` = 13.500 V, which is a
  believable resting battery. A wrong shift gives 3456 V or 0.05 V.
- **Set equality, not spot checks.** "Group 0x0A nodes are exactly the
  extended-frame nodes" was tested as a set comparison across the whole capture.
- **Disproving a source on its own data.** As above with the UnityX parser.
- **Range guards in shipped code.** Where a conclusion is unverified, the
  firmware rejects implausible values and logs instead of publishing. A wrong
  guess then shows up as a warning rather than as a believable number.

## What the bus turned out to be

### Frame layout

Broadcast, 11-bit: `ID = (MESSAGE_TYPE << 8) | source_address`, with
`MESSAGE_TYPE` at `a0079:4028-4092` (`COMMAND = 130` at `a0079:4044`).

| Type | Name | Used here for |
|---|---|---|
| 0 | NETWORK | nothing |
| 1 | CIRCUIT_ID | all zero on this coach |
| 2 | DEVICE_ID | the device inventory |
| 3 | DEVICE_STATUS | tank levels, light and relay state |
| 6 | PRODUCT_STATUS | single byte, nothing useful |
| 7 | TIME | master clock, 1 Hz |

Point-to-point, 29-bit:

```
ID = (type & 0x1C) << 24 | src << 18 | (type & 3) << 16 | dst << 8 | message_data
```

with `REQUEST = 128`, `RESPONSE = 129`, `COMMAND = 130`, `EXT_STATUS = 131`.
This formula was accepted only after it reproduced every captured app frame.

All multi-byte fields are big-endian. See `GetUINT16` at `a0078:5025`.

### Self-discovery: the device inventory

Every node broadcasts its own identity on `0x200 | address` once a second.
Payload layout from `a0079:5444`:

```
[0..1] PRODUCT_ID      [2] product instance
[3]    DEVICE_TYPE     [4..5] FUNCTION_NAME
[6]    device instance << 4 | function instance
[7]    device capabilities
```

Decoding that against `DEVICE_TYPE` (`a0080:600-780`) and `FUNCTION_NAME`
(`a0080:903-2328`) turns the bus into a self-describing inventory. This is where
every address in `examples/coach.yaml` comes from. None were guessed.
`discover.yaml` is that decoder, reduced to the parts worth logging.

The eight names in the OneControl app matched eight nodes with no leftovers on
either side. The check that this was right: the Soffit, which had been found
the hard way weeks earlier by toggling it and watching the wire, landed on the
same address the decoder independently produced.

### Sessions: why replaying a command does nothing

A device accepts a command only from the client holding an open
`REMOTE_CONTROL` session. From the receive guard at `a0079:25211`:

```csharp
rx.SourceAddress == GetLocalSessionClientAddress(SESSION_ID 4)
```

Opening one is a seed/key challenge: request a seed, transform it, send back the
key. The transform is a modified TEA, 32 rounds, keyed on the session's own
constant, `SESSION_ID(4, 2976579765u, "REMOTE_CONTROL", ...)` at `a0080:3727`,
where `2976579765` is `0xB16B00B5`.

Confirmed by reproducing **both** captured seed/key pairs from the app's own
handshakes exactly. Two independent pairs matching is not coincidence.

The config opens and closes a session per command, which is what the app does.
Nothing is held, so the phone app is never locked out and no heartbeat is owed.

### Commands: two device classes, opposite conventions

This is the part that caused the worst bug, and it is worth stating plainly.

**Dimmable lights** put the command in an **8-byte payload** and leave message
data at 0. See `LogicalDeviceLightDimmableCommand`, `d0165:16412`. Byte 0 is the
mode, byte 1 is brightness 1-100.

**Latching relays** do the exact reverse: the command rides in the **message
data byte of the ID**, and the payload must be **empty**. From the relay's own
receive handler at `a0079:25211`:

```csharp
if (rx.MessageType == 130 && rx.TargetAddress == Address
    && rx.SourceAddress == <session client>
    && rx.Count == 0)          // payload MUST be empty
    Command = rx.MessageData;  // command travels in the ID
```

`COMMAND_MODE`: `OFF = 0`, `ON = 1`, `CLEAR_LATCH = 3`.

### Motors: the same convention, plus a deadman

Nothing in this repository drives a motor, and none of what follows is
implemented here. It is recorded because it is part of how the bus was worked
out, and because knowing what a motor node expects is how you know to leave it
alone.

The awning motors are `DEVICE_TYPE 33 MOMENTARY_H_BRIDGE_TYPE_2`, and since
that class derives from `RELAY_TYPE_2` it inherits the relay convention exactly:
`MessageType 130`, **empty payload**, command in the message-data byte. Only the
value set is wider (`a0079:28109`):

```csharp
public enum COMMAND_MODE : byte
{ STOP, FORWARD, REVERSE, CLEAR_OUTPUT_DISABLED_LATCH, HOME_RESET,
  AUTO_FORWARD, AUTO_REVERSE }
```

`FORWARD` is documented by the output state it produces:
`FORWARD_EXTEND_CLOCKWISE_OUT_UP`. So 1 extends and 2 retracts, and the naming
removes any need to guess which way is out.

Two things about this class do not apply to anything else on the bus.

**The motor only runs while you hold the session.** Ending it is itself a stop
(`a0079:28301`, inside `MOMENTARY_H_BRIDGE_TYPE_2_SIM_INTERFACE`, which starts
at 28107; `MOMENTARY_RELAY_TYPE_2` at 28479 has a near-identical check for its
own `ON` state, so the class matters as much as the line):

```csharp
else if (rx.MessageType == 128 && (REQUEST)rx.MessageData == 69   // SESSION_END
         && GetLocalSession(4)?.Client?.Address == rx.SourceAddress
         && (OutputState == FORWARD_EXTEND... || OutputState == REVERSE_RETRACT...))
    OutputState = OFF_STOP;
```

That is a deliberate deadman, and it matches the app's press-and-hold UI. It
also means the light path's fire-and-forget shape (open, command, close, all
inside 100 ms) would move an awning for 50 ms and stop. Motion needs a *held*
session with a heartbeat, which is a different shape entirely from anything in
this repository.

**The session lapses after five seconds of silence** (`a0079:14791`,
`SESSION_TIMEOUT = TimeSpan.FromSeconds(5.0)`), and when it does the node sets
its client address to `INVALID` and stops accepting our commands. The failure
mode worth engineering against is therefore not "the motor keeps running", it is
"the motor keeps running and we can no longer tell it to stop". Anything holding
a session for motion wants a heartbeat well inside that window: 1 s leaves four
missed beats of margin.

Note also that `AUTO_FORWARD` and `AUTO_REVERSE` exist in the enum but the
`Command` setter's switch only handles 0-3. They are unimplemented in this
class, so sending them does nothing. Presence in an enum is not a feature.

### Status decoding

Relay status, from the device's own `UpdateStatus` at `a0079:25022`: byte 0 low
nibble is output state, bit 5 latch, bits 6-7 command-allowed; byte 1 position;
bytes 2-3 current draw as 8.8 fixed point; bytes 4-5 DTC. Cross-checked against
the capture, where all four relays read `80:FF:00:01:00:00`.

Tank level is byte 0 of `DEVICE_STATUS`, percent full. These report in 33%
steps: a three-segment probe, not a precise third.

### PID reads: asking for what is not broadcast

Voltage is not broadcast by anything. It is fetched with `PID_READ_WRITE`
(REQUEST 17): send the PID as a big-endian `uint16`, and the reply echoes it and
appends the value.

`PID.BATTERY_VOLTAGE = 43` at `a0080:2721`, formatted `UINT32` scaled by
`1.52587890625E-05`, which is 1/65536, i.e. 16.16 fixed point.

Verified against UnityX rather than assumed:

```
0x02145C11 #00:2B          -> read PID 43
0x01718511 #00:2B:0D:7F:F3 -> 0x0D7FF3 / 65536 = 13.500 V
```

Note the reply carries **three** value bytes, not the four a `UINT32` implies.
The leading zero is simply not sent. Width therefore comes from the DLC.

Reads need no session; the `write_SessionId` in the PID constructor
(`a0080:3566`) governs writes only.

## Wrong turns

The instructive part. Every one of these shipped or nearly shipped.

### Calling the session handshake "generic bus chatter"

The seed/key frames were spotted early and correctly flagged as a possible
rolling code. Then, after seeing similar frames in the UnityX capture, they were
downgraded to unrelated chatter and that guess was written into a config
comment as if settled.

They were the session handshake. Commands were being sent with no session and
silently dropped.

**Lesson:** finding a pattern elsewhere is not evidence it is unimportant.

### "0.3 gallons": right PID, wrong byte, and eventually abandoned

Tank capacity came back as `1`, so a 33% tank reported 0.3 gal.

The PID and transport were correct. Capacity lives in `ACCESSORY_SETTING_02`
(`a0080:3327`), a **6-byte** record, and the first of those six bytes was taken
as a whole number of gallons. The app extracts it with `BitPositionValue64`
(mask `0xFF`, start bit 0) at `d0165:33973` from the value assembled as an
integer, and since bytes accumulate big-endian, bit 0 is the **last** byte.

Also dropped `TANK_CAPACITY_GALLONS` (PID 406, `a0080:3447`) from the poll: it
is the obvious candidate by name but is defined and never referenced anywhere in
the app, while 346 is what `TankCapacityPid` actually wraps (`d0165:32594`).

The corrected last-byte version did not work either, and gallons have since
been removed entirely. Percentages are read straight from `DEVICE_STATUS` and
have been reliable throughout; it is only the *capacity* that could never be
read off this coach. Anybody wanting gallons is better off multiplying by a
number from the spec sheet than continuing to argue with PID 346.

**Lesson:** a plausible number is more dangerous than an error, and knowing
when to stop is part of the method. Two failed attempts on a value that can be
typed in by hand is the point at which the bus read stops being worth it.

### Every relay silently ignoring us

The worst one, because it shipped and looked fine. The relay command values
(`Off = 0`, `On = 1`) were found in the app's *sending* code and assumed to be
payload bytes. The `rx.Count == 0` requirement lives in the *receiving* code,
which had not been read.

Result: `rx.Count == 1`, guard fails, command discarded with no error. All six
relays were affected; only two had been tried, so it looked like a two-device
problem and the first hypothesis offered was that the addresses were wrong. They
were not.

**Lesson:** when transmitting, read the receiver's validation, not the sender's
construction.

### An enum collision that nearly lost the water pump

The water pump decoded as "DAQ". Two unrelated constant tables in the firmware
both define `5`, and a loose text scrape matched the wrong one. Scoping the
lookup to the `FUNCTION_NAME` class (`a0080:903-2328`) resolves it to
`WATER_PUMP`.

Re-running all 36 devices through the corrected table left every light
assignment unchanged, which is the only reason to believe the rest of the map.

**Lesson:** grepping a decompile for `NAME = 5` is not the same as reading the
enum that owns it.

### Device class is not fixture capability

`0xBC` is a `DIMMABLE_LIGHT`, so it got a brightness slider. The fixture strobes
on anything below full. Kitchen Pendants, same device class, dims properly.

**Lesson:** the firmware describes the driver. Only the coach knows what is
wired to it.

### Two ESPHome defaults that would have been quietly wrong

Not protocol issues, but the same category of error:

- `default_transition_length` is 1 s, and a transition re-runs the output every
  step. Each step here is a full session handshake plus command frame, so one
  slider drag would have put dozens of sessions on the bus. Set to 0 s.
- `gamma_correct` defaults to 2.8. The node applies its own curve, so correcting
  twice would have made 50% arrive as 14. Set to 1.0.

## Still unverified

Stated plainly, because the above is a long argument for doing so:

- **Tank capacity, entirely.** Two readings of PID 346 were attempted, first
  byte and last byte, and neither produced a sane number on this coach. Gallons
  are gone; tank percentages are unaffected and reliable.
- **Whether the awnings dim.** Same device class as the Soffit, untested.
- **The awning motors in motion.** Their addresses, device class and stopped
  status decode are confirmed against the capture, and the command format and
  deadman come from the firmware. But the capture only ever caught both motors
  at rest (`C0:FF:00:00:00:00`), so the moving output states, the current-draw
  reading under load and the end-of-travel flags flipping have all been read
  and none of them have been seen.
- **Whether real awning hardware has its own motor watchdog.** The class read
  here is the app's *simulator* of the device, which stops on `SESSION_END` and
  on an explicit `STOP` but has no timer of its own. Physical firmware very
  likely does. Nothing here relies on it.
- **Three unnamed tank sensors** at `0x2E`, `0xC6`, `0xD4` (`FUNCTION_NAME` 0).
  Two set a fuel-sensor capability bit. Left unexposed.
- **Which physical grey tank is 1, 2 or 3.** The bus reveals only the function
  instance.
- **The rest of the coach.** Slides, leveler, generator and hour meter are all
  identified in the inventory and none are exposed. The three slides are the
  same `MOMENTARY_H_BRIDGE_TYPE_2` class as the awnings and would work the same
  way, which is precisely why they have been left alone.

## Reproducing this

Add `discover.yaml` to your `files:` list, flash, and run:

```
esphome logs your-node.yaml
```

The `oc.discover` lines are the decoded device inventory, one per node. Going
further than that means adding your own `on_frame` handler to log raw frames;
this repository does not ship one, because a coach at 1 Hz across 36 nodes fills
a log fast. Toggle one thing at a time and watch what moves.
