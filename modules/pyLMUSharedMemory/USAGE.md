# pyLMUSharedMemory — Usage Guide

A comprehensive, implementation-verified guide to using pyLMUSharedMemory to
read real-time telemetry, scoring, event, path, and application-state data from
Le Mans Ultimate (LMU).

## Table of contents

- [Overview](#overview)
- [Getting started and recipes](#getting-started-and-recipes)
  - [Installation and import path](#1-installation-and-import-path)
  - [Choose a reader](#2-choose-a-reader)
  - [Quick start: open, update, read, close](#3-quick-start-open-update-read-close)
  - [Decide whether data is usable](#4-decide-whether-data-is-usable)
  - [Decode text fields](#5-decode-text-fields)
  - [Common recipes](#common-recipes)
  - [Access-mode tradeoffs](#access-mode-tradeoffs)
  - [Troubleshooting](#troubleshooting)
- [API reference](#api-reference)
  - [Package exports](#package-exports)
  - [Mapping and reader API](#lmu_mmap-mapping-and-reader-api)
  - [Constants, structures, and legacy reader](#lmu_data-constants-structures-and-legacy-reader)
  - [Enum lookup API](#lmu_enum-enum-lookup-api)
  - [Annotation-only classes](#lmu_type-annotation-only-classes)
  - [Repository diagnostics](#repository-diagnostic-functions)
- [Enum reference](#enum-reference)
  - [Vehicle identity](#vehicle-identity)
  - [Session and race state](#session-and-race-state)
  - [Vehicle controls and hardware](#vehicle-controls-and-hardware)
  - [Wheel, tyre, and track conditions](#wheel-tyre-and-track-conditions)
- [Complete shared-memory data reference](#complete-shared-memory-data-reference)
  - [Type and validity conventions](#type-and-validity-conventions)
  - [Root layout](#root-layout)
  - [Generic, event, application, and path data](#generic-event-application-and-path-data)
  - [Scoring data](#scoring-data)
  - [Telemetry data](#telemetry-data)
  - [Vector type](#vector-type)
  - [Structure size checklist](#structure-size-checklist)

---

## Overview

Le Mans Ultimate publishes a fixed binary shared-memory block named
`LMU_Data`. pyLMUSharedMemory maps that block to Python `ctypes` structures,
so projects can read the game's values using the original interface field
names.

The module provides:

- Live physics and controls for as many as 104 vehicles, including engine,
  hybrid, aerodynamic, damage, setup, gap, and per-wheel values.
- Session scoring, standings, lap and sector times, flags, pit state, weather,
  track state, and multiplayer server details.
- Game events, version, force-feedback torque, window/UI state, and game paths.
- Strict enum classes plus a tolerant lookup helper.
- A stable snapshot reader and a legacy Windows live-view reader.

The library only reads/exposes the mapping; it does not calculate strategy,
persist history, send controls to LMU, or turn values into JSON. The tables in
this guide distinguish facts defined in the source from units or semantics that
the current repository does not specify.

---

## Getting started and recipes

## 1. Installation and import path

The repository is a Python package directory but does not contain packaging
metadata. Clone it, then make its parent directory importable:

```console
git clone https://github.com/dicarlocolin/pyLMUSharedMemory.git
```

For a one-off shell, set `PYTHONPATH` to the directory containing the clone.
For a normal project, keeping your project and the clone under the same parent
is sufficient when the program is launched from that parent. Test the path with:

```console
python -c "from pyLMUSharedMemory import lmu_data; print(lmu_data.LMUConstants.LMU_SHARED_MEMORY_FILE)"
```

The result should be `LMU_Data`. A `ModuleNotFoundError` normally means the
clone itself, rather than its parent, is the only import directory.

## 2. Choose a reader

| Reader | Platform path | Snapshot support | Recommendation |
|---|---|---|---|
| `lmu_mmap.MMapControl` | Windows named mapping; Linux `/dev/shm/LMU_Data` | Copy mode is default; direct mode optional | Use for new projects. |
| `lmu_data.SimInfo` | Windows named mapping only | No; always a live view | Use only for compatibility with existing code. |

Copy mode is the safest default when a screen, network response, or log record
depends on several values agreeing with each other. Direct mode avoids a
324,820-byte copy per read cycle, but LMU may update between individual reads.

## 3. Quick start: open, update, read, close

```python
from pyLMUSharedMemory import lmu_data, lmu_mmap

reader = lmu_mmap.MMapControl(
    lmu_data.LMUConstants.LMU_SHARED_MEMORY_FILE,
    lmu_data.LMUObjectOut,
)
reader.create()  # copy mode

try:
    reader.update()
    root = reader.data
    print(root.generic.gameVersion)
finally:
    reader.close()
```

In copy mode, an update is accepted only when an update event is truthy and
the scoring and telemetry active-vehicle counts agree. Until then, `data`
remains the last accepted snapshot. The initial snapshot is copied at
`create()` time without that check.

## 4. Decide whether data is usable

The mapping helpers can create an empty mapping when the publisher is absent,
so successful `create()` does not prove that LMU is active. The existing sample
code uses `generic.gameVersion != 0` as its basic readiness signal.

Before selecting the player, also require `playerHasVehicle` and validate the
index against both the active count and capacity:

```python
telemetry = reader.data.telemetry
limit = min(
    telemetry.activeVehicles,
    lmu_data.LMUConstants.MAX_MAPPED_VEHICLES,
)

if telemetry.playerHasVehicle and telemetry.playerVehicleIdx < limit:
    player = telemetry.telemInfo[telemetry.playerVehicleIdx]
else:
    player = None
```

Shared memory is external mutable input. Bounds checks are worthwhile even for
fields that are normally valid.

## 5. Decode text fields

Fixed C text arrays are returned as `bytes`, typically ending at the first NUL.
The interface does not declare an encoding in this repository. UTF-8 with
replacement is a robust display default; use the encoding appropriate to your
environment if names are encoded differently.

```python
def decode_lmu(raw: bytes) -> str:
    return raw.decode("utf-8", errors="replace")


track = decode_lmu(reader.data.scoring.scoringInfo.mTrackName)
driver = decode_lmu(
    reader.data.scoring.vehScoringInfo[0].mDriverName
)
```

Byte-array placeholders such as `mExpansion`, and arrays representing opaque
pointers, are not text and should not be decoded.

## Common recipes

### Complete realtime telemetry loop

This complete example opens copy mode, waits for published game data, validates
the player index, displays a few core values at roughly 20 Hz, and closes cleanly
on Ctrl+C:

```python
import time

from pyLMUSharedMemory import lmu_data, lmu_mmap


def decode_lmu(raw: bytes) -> str:
    return raw.decode("utf-8", errors="replace")


reader = lmu_mmap.MMapControl(
    lmu_data.LMUConstants.LMU_SHARED_MEMORY_FILE,
    lmu_data.LMUObjectOut,
)
reader.create(access_mode=0)

try:
    while True:
        reader.update()
        root = reader.data
        telemetry = root.telemetry

        active = min(
            telemetry.activeVehicles,
            lmu_data.LMUConstants.MAX_MAPPED_VEHICLES,
        )
        index = telemetry.playerVehicleIdx

        if (
            root.generic.gameVersion == 0
            or not telemetry.playerHasVehicle
            or index >= active
        ):
            print("\rWaiting for an active player vehicle...", end="", flush=True)
            time.sleep(0.25)
            continue

        player = telemetry.telemInfo[index]
        velocity = player.mLocalVel
        speed_kph = (
            velocity.x**2 + velocity.y**2 + velocity.z**2
        ) ** 0.5 * 3.6
        gear = "R" if player.mGear == -1 else (
            "N" if player.mGear == 0 else str(player.mGear)
        )

        print(
            f"\r{decode_lmu(player.mVehicleName):<30} "
            f"{speed_kph:6.1f} km/h  "
            f"gear {gear:>2}  "
            f"{player.mEngineRPM:7.0f} rpm  "
            f"fuel {player.mFuel:5.1f} L",
            end="",
            flush=True,
        )
        time.sleep(0.05)
except KeyboardInterrupt:
    print()
finally:
    reader.close()
```

### Iterate active scoring and telemetry records

Scoring and telemetry have separate active counts. Slice syntax on a `ctypes`
array produces a Python list, but an explicit range avoids an allocation:

```python
root = reader.data

scoring_count = min(
    root.scoring.scoringInfo.mNumVehicles,
    lmu_data.LMUConstants.MAX_MAPPED_VEHICLES,
)
for index in range(scoring_count):
    score = root.scoring.vehScoringInfo[index]
    print(score.mID, score.mPlace, decode_lmu(score.mDriverName))

telemetry_count = min(
    root.telemetry.activeVehicles,
    lmu_data.LMUConstants.MAX_MAPPED_VEHICLES,
)
for index in range(telemetry_count):
    car = root.telemetry.telemInfo[index]
    print(car.mID, car.mEngineRPM, car.mFuel)
```

### Join scoring and telemetry safely

Use `mID`, not a saved array index, to correlate the two record types. IDs can
be reused after a participant leaves, so a long-lived identity should combine
the slot ID with other identity fields and detect roster changes.

```python
root = reader.data
score_count = min(root.scoring.scoringInfo.mNumVehicles, 104)
telem_count = min(root.telemetry.activeVehicles, 104)

score_by_id = {
    root.scoring.vehScoringInfo[i].mID:
        root.scoring.vehScoringInfo[i]
    for i in range(score_count)
}

for i in range(telem_count):
    car = root.telemetry.telemInfo[i]
    score = score_by_id.get(car.mID)
    if score is not None:
        print(score.mPlace, decode_lmu(score.mDriverName), car.mEngineRPM)
```

### Read wheel data

Wheel order is fixed: front-left, front-right, rear-left, rear-right.

```python
from pyLMUSharedMemory.lmu_enum import LMUWheelIndex

for index, wheel in enumerate(player.mWheels):
    temperatures_c = [value - 273.15 for value in wheel.mTemperature]
    print(
        LMUWheelIndex(index).name,
        f"pressure={wheel.mPressure:.1f} kPa",
        f"surface={temperatures_c}",
        f"brake={wheel.mBrakeTemp:.1f} C",
    )
```

`mTemperature` is left/center/right, not inside/center/outside. This distinction
matters because "inside" reverses between left and right sides of the car.

### Convert enum-backed values

Direct construction is convenient when the producer and reader versions are
known to agree:

```python
from pyLMUSharedMemory import lmu_enum

session = lmu_enum.LMUSession(root.scoring.scoringInfo.mSession)
print(session.name)
```

For a long-running integration that should tolerate a future unknown value,
build a fallback mapper once:

```python
session_name = lmu_enum.enum_map(lmu_enum.LMUSession)
print(session_name(root.scoring.scoringInfo.mSession))
```

`mYellowFlagState` is an unusual exception: its runtime value is a one-byte
`bytes` object because the structure declares `ctypes.c_char`. Convert it to a
signed integer before using `LMUYellowFlagState`:

```python
raw = root.scoring.scoringInfo.mYellowFlagState
yellow_value = int.from_bytes(raw, byteorder="little", signed=True)
yellow = lmu_enum.LMUYellowFlagState(yellow_value)
```

### Calculate speed

The interface exposes local velocity components in metres per second. Vector
magnitude is direction-independent:

```python
velocity = player.mLocalVel
speed_mps = (velocity.x**2 + velocity.y**2 + velocity.z**2) ** 0.5
speed_kph = speed_mps * 3.6
speed_mph = speed_mps * 2.2369362921
```

Do not assume one component alone is always forward speed unless you have
validated LMU's axis convention for the interface version you target.

### Calculate remaining fuel percentage

Telemetry provides litres and capacity directly:

```python
fuel_percent = (
    player.mFuel / player.mFuelCapacity * 100.0
    if player.mFuelCapacity > 0.0
    else None
)
```

Scoring also exposes `mFuelFraction` as an encoded byte where `0x00` means 0%
and `0xFF` means 100%:

```python
fuel_percent_from_scoring = score.mFuelFraction / 255.0 * 100.0
```

Do not divide that byte by 100.

### Interpret the telemetry current sector

`LMUVehicleTelemetry.mCurrentSector` is a zero-based sector with the pit-lane
state stored in the sign/high bit. Treat it as an unsigned 32-bit bit pattern:

```python
raw_sector = player.mCurrentSector & 0xFFFFFFFF
in_pit_lane = bool(raw_sector & 0x80000000)
sector_zero_based = raw_sector & 0x7FFFFFFF
```

This field differs from `LMUVehicleScoring.mSector`, whose values are the
non-linear `0 = sector 3`, `1 = sector 1`, `2 = sector 2` mapping.

### Save raw bytes for debugging

The legacy reader has a `save()` helper:

```python
from pyLMUSharedMemory.lmu_data import SimInfo

info = SimInfo()
try:
    info.save("LMU_SHARED_MEMORY_FILE.bin")
finally:
    info.close()
```

With `MMapControl`, make a normal binary copy of the detached/snapshot structure
using standard `ctypes` only if your application needs it; the project does not
provide a second save method. Raw captures may contain player names, Steam IDs,
server details, and local filesystem paths, so handle or share them accordingly.

## Access-mode tradeoffs

| Concern | Copy mode (`0`) | Direct mode (`1`) |
|---|---|---|
| Cross-field consistency | Better: reads use the last accepted full snapshot. | Weaker: producer can update during reads. |
| Per-update work | Copies 324,820 bytes when the guard passes. | No copying. |
| Freshness | Changes appear only after `update()`, and may remain stale while the guard fails. | Each field read sees the mapping at that instant. |
| Closing | `data` already uses a private buffer; close still replaces it with a final copy. | Release nested/live references to reduce `BufferError` risk. |
| Typical use | Dashboards, logging, APIs, analysis. | Very tight polling where occasional mixed frames are acceptable. |

## Troubleshooting

### All fields remain zero

- Confirm LMU is running and has entered a state where it publishes the mapping.
- Check `generic.gameVersion`; the examples use zero to mean not ready.
- On Linux, remember that `linux_mmap()` may create a zero-filled
  `/dev/shm/LMU_Data` itself.
- In copy mode, inspect whether the event indicators become truthy and whether
  scoring and telemetry vehicle counts agree; otherwise `update()` keeps the
  previous snapshot.

### `ModuleNotFoundError: pyLMUSharedMemory`

Run from, or add to `PYTHONPATH`, the directory containing the repository. The
repository directory itself is the package, not the import root.

### `TypeError` mentioning `tagname`

`SimInfo` was used on a non-Windows `mmap` implementation. Switch to
`MMapControl`; live LMU data still requires a publisher compatible with that
platform backend.

### `ValueError` when converting an enum

The raw producer value is not a member of the local enum (or, for
`mYellowFlagState`, it has not yet been converted from `bytes`). Use the
documented conversion or `enum_map(EnumClass)(value)` for a fallback string.

### `BufferError` while closing

Drop references to live `ctypes` objects obtained from a direct mapping before
calling `close()`. Prefer copy mode when data needs to survive independently of
the OS mapping.

### Data looks stale in copy mode

Call `update()` every polling cycle. If it still does not change, the snapshot
guard may be rejecting copies because neither update event is truthy or the two
active-vehicle counts differ. Direct mode can help diagnose the producer state,
but its reads can be internally inconsistent.

---

## API reference

This page documents every module-defined class, function, and method intended
to be reachable by a project. The shared-memory fields exposed through the
reader are listed in the
[complete shared-memory data reference](#complete-shared-memory-data-reference),
and the enum members are in the [enum reference](#enum-reference).

## Package exports

The package's `__init__.py` is empty. Nothing is re-exported directly from
`pyLMUSharedMemory`, so import the submodules you need:

```python
from pyLMUSharedMemory import lmu_data, lmu_enum, lmu_mmap, lmu_type
```

`lmu_data` contains the real `ctypes` structures and legacy reader;
`lmu_mmap` contains the recommended reader; `lmu_enum` contains value mappings;
and `lmu_type` contains non-instantiable annotation-only mirrors.

## `lmu_mmap`: mapping and reader API

### Module constants

| Name | Type/value | Meaning |
|---|---:|---|
| `PLATFORM` | `str` | Result of `platform.system()` captured when `lmu_mmap` is imported. Used to select the mapping backend. |
| `MAX_VEHICLES` | `104` | Alias of `LMUConstants.MAX_MAPPED_VEHICLES`. |
| `INVALID_INDEX` | `-1` | Conventional invalid-index sentinel. The current reader does not use it internally. |
| `logger` | `logging.Logger` | Logger used by `MMapControl`; its name is selected by `get_root_logger_name()`. |

### `MMapControl(mmap_name, data_struct)`

The recommended mapping lifecycle controller.

Parameters:

| Parameter | Expected value | Meaning |
|---|---|---|
| `mmap_name` | `str` | Named mapping on Windows, or filename below `/dev/shm` on Linux. Use `LMUConstants.LMU_SHARED_MEMORY_FILE`. |
| `data_struct` | `ctypes.Structure` subclass | Structure used to interpret the mapping. Use the class `LMUObjectOut`, not an instance. |

Public attributes:

| Attribute | State and meaning |
|---|---|
| `data` | `None` before `create()`. Afterwards, an instance of `data_struct`: either a live mapping view (direct mode) or a stable byte-buffer view (copy mode). After `close()`, it is a final detached copy. |
| `update` | `None` before `create()` and after `close()`. Between them, a zero-argument callable selected for the access mode. |

#### `create(access_mode=0) -> None`

Opens or creates a platform mapping whose byte length is
`ctypes.sizeof(data_struct)` and initializes `data` and `update`.

| `access_mode` | Mode | `data` behavior | `update()` behavior |
|---:|---|---|---|
| `0` (default, or any false value) | Copy | Views an internal `bytearray` snapshot. A group of reads stays stable until the next successful update. | Copies the complete mapping only when either scoring or telemetry update indicator is truthy **and** `scoringInfo.mNumVehicles == telemetry.activeVehicles`. If that consistency check is false, the previous snapshot remains available. |
| Any truthy value (normally `1`) | Direct | Views the shared mapping itself. Values can change while the caller reads related fields. | No-op; the live view needs no refresh. |

Call `create()` exactly once for an open/consume/close lifecycle. The method has
no explicit guard against opening the same controller twice.

#### `close() -> None`

Copies the current mapping into a detached structure, assigns that copy to
`data`, releases the internal realtime view, tries to close the OS mapping, and
sets `update` to `None`. This preserves one last readable snapshot.

Call it only after a successful `create()`. Holding extra direct `ctypes` views
or references into the mapping can make `mmap.close()` raise `BufferError`; the
method catches that error and logs it, so release such references before close.

#### Destructor and private update methods

`MMapControl.__del__()` only logs that the controller was garbage-collected; it
does **not** call `close()`. Use an explicit `try/finally`.

`__buffer_share()` and `__buffer_copy()` are name-mangled implementation
methods assigned to `update`; they are not application API. The first is the
direct-mode no-op, and the second performs the guarded copy described above.

### Mapping helper functions

| Function | Return | Behavior and caveats |
|---|---|---|
| `platform_mmap(name, size)` | `mmap.mmap` | Calls `windows_mmap` when the import-time `PLATFORM` equals `"Windows"`; otherwise calls `linux_mmap`. |
| `windows_mmap(name, size)` | `mmap.mmap` | Creates/opens a named, anonymous mapping using `mmap.mmap(-1, size, name)`. `name` should be `"LMU_Data"`; `size` should be `ctypes.sizeof(LMUObjectOut)`. |
| `linux_mmap(name, size)` | `mmap.mmap` | Opens `/dev/shm/<name>` in `a+b` mode, creating it if allowed. If its initial file position is zero, writes `size` zero bytes, then maps `size` bytes. A newly created zero-filled file is not evidence that the game is running. |
| `get_root_logger_name()` | `str` | Returns the first key currently found in `logging.root.manager.loggerDict`; if there is none, returns `lmu_mmap`'s module name. Because this depends on logger registration order, configure the exported `logger` directly when deterministic logging is needed. |
| `test_api()` | `None` | Console demonstration: enables INFO logging, opens direct mode, closes it, opens copy mode, prints game/track/car/count values, then closes. It needs a usable mapping and is intended for manual diagnostics. |

The low-level mapping helpers are public Python names, but most applications
should let `MMapControl.create()` call them.

## `lmu_data`: constants, structures, and legacy reader

### `LMUConstants`

| Constant | Value | Meaning |
|---|---:|---|
| `LMU_SHARED_MEMORY_FILE` | `"LMU_Data"` | Mapping tag/file published by the interface. |
| `LMU_PROCESS_NAME` | `"Le Mans Ultimate"` | Human-readable process name. No current method performs process discovery with it. |
| `MAX_MAPPED_VEHICLES` | `104` | Fixed capacity of both per-vehicle arrays. Use active counts to select valid entries. |
| `MAX_PATH_LENGTH` | `260` | Fixed C buffer length for each path field, matching the traditional Windows maximum path length. |

### `ctypes.Structure` classes

All real data classes use `_pack_ = 4` to match the game's four-byte packing.
They contain declarations rather than custom application methods.

| Class | C mapping/purpose | Size |
|---|---|---:|
| `LMUVect3` | `TelemVect3`, a three-component vector | 24 bytes |
| `LMUWheel` | `TelemWheelV01`, one wheel | 260 bytes |
| `LMUVehicleTelemetry` | `TelemInfoV01`, one vehicle's physics/controls | 1,888 bytes |
| `LMUVehicleScoring` | `VehicleScoringInfoV01`, one vehicle's timing/race state | 584 bytes |
| `LMUScoringInfo` | `ScoringInfoV01`, session-wide state | 548 bytes |
| `LMUApplicationState` | `ApplicationStateV01`, game window/UI state | 260 bytes |
| `LMUScoringData` | Session scoring wrapper and 104 records | 126,832 bytes |
| `LMUTelemetryData` | Telemetry wrapper and 104 records | 196,356 bytes |
| `LMUPathData` | Five installation/user paths | 1,300 bytes |
| `LMUEvent` | Sixteen shared-memory event indicators | 64 bytes |
| `LMUGeneric` | Events, version, FFB, application state | 332 bytes |
| `LMUObjectOut` | Root object read by applications | 324,820 bytes |
| `LMULayout` | One-field wrapper around `LMUObjectOut` | 324,820 bytes |

Because these inherit `ctypes.Structure`, standard operations such as
`ctypes.sizeof(Type)`, `Type.from_buffer(buffer)`, and
`Type.from_buffer_copy(buffer)` are available. They are standard-library APIs,
not custom methods from this project. Constructing a structure with `Type()`
creates zeroed local memory; it does not connect to LMU.

### `SimInfo()`

Legacy Windows-only convenience reader. Construction immediately opens a
`LMU_Data` named mapping of `sizeof(LMUObjectOut)` and assigns a live structure
view to `LMUData`.

| Attribute | Meaning |
|---|---|
| `_lmu_data` | Underlying `mmap.mmap`; internal implementation detail. |
| `LMUData` | Live `LMUObjectOut` view. It is set to `None` by `close()`. |

#### `save(filename) -> None`

Writes a byte-for-byte snapshot of the complete 324,820-byte mapping to
`filename` in binary mode, replacing an existing file. This is useful for raw
capture/ABI debugging. The method does not serialize field names or JSON.

#### `close() -> None`

Sets `LMUData` to `None` and attempts to close the underlying mapping. If an
exported buffer reference remains, it catches `BufferError` and prints the
error to stdout. It does not return a final data snapshot.

#### `__del__()`

Calls `close()` during garbage collection. Destruction timing is not reliable,
so application code should still call `close()` explicitly.

`SimInfo` directly passes the Windows-specific `tagname` keyword to `mmap` and
will raise `TypeError` on platforms whose Python `mmap` does not accept it. It
also presents a live view with no consistency/snapshot mechanism.

## `lmu_enum`: enum lookup API

### `enum_map(reference, default="Unknown")`

Builds a dictionary from each enum member's `.value` to `.name`, then returns a
one-argument lookup function.

```python
session_name = lmu_enum.enum_map(lmu_enum.LMUSession)
print(session_name(10))   # Race1
print(session_name(99))   # Unknown

strict = lmu_enum.LMUSession(10)
print(strict.name)        # Race1
```

| Parameter | Meaning |
|---|---|
| `reference` | Iterable of enum members, normally one of this module's `Enum` classes. |
| `default` | String returned for any key absent from the generated mapping. |

The returned closure accepts an integer-like lookup key and returns a member
name or `default`. Direct enum construction is strict and raises `ValueError`
for an unknown value; the closure is forward-compatible with new game values.
The closure captures a snapshot, so later enum changes would require rebuilding
it.

### Enum classes and `test()`

The module provides 20 `enum.Enum` subclasses. See every member and its related
field in the [enum reference](#enum-reference). `lmu_enum.test()` is a
console-only demonstration of direct construction and `enum_map()` lookups; it
prints sample values and returns `None`.

## `lmu_type`: annotation-only classes

`lmu_type` mirrors the field names from `lmu_data` with Python-oriented type
annotations. These classes inherit the abstract `_NOINIT` base and are not live
data structures. Do not construct them, pass them to `MMapControl`, or use
`ctypes.sizeof()` on them.

They are useful only as annotation references in editors/type checkers. Runtime
objects come from the same-named classes in `lmu_data`:

```python
from typing import TYPE_CHECKING

from pyLMUSharedMemory import lmu_data

if TYPE_CHECKING:
    from pyLMUSharedMemory.lmu_type import LMUVehicleTelemetry


def rpm(car: "LMUVehicleTelemetry") -> float:
    return car.mEngineRPM


car = lmu_data.LMUVehicleTelemetry()  # runtime ctypes object
```

The annotation module currently describes the intended Python-facing types but
is not authoritative for byte layout. In particular, fixed `ctypes` arrays are
array objects at runtime (not tuples), `LMUScoringData.scoringStreamSize` is a
12-byte array in the implementation, and `LMUScoringInfo.mYellowFlagState` is a
one-byte `bytes` value because its field type is `ctypes.c_char`.

## Repository diagnostic functions

`tests/read_lmu_api.py` is a manual diagnostic program rather than package API,
but every function in it is summarized here for maintainers:

| Function | Purpose |
|---|---|
| `verify_struct_size(s_class, size_origin)` | Prints actual/expected sizes and raises `ValueError` on mismatch. |
| `compare_struct_size()` | Checks all 13 structure sizes against the expected LMU layout. |
| `event_info(data)` | Prints a selected subset of event indicators. |
| `generic_info(data)` | Prints version, FFB, window, and options-page state. |
| `path_info(data)` | Prints all five path buffers. |
| `scoring_info(data)` | Prints selected session/scoring/weather values and converts grip/cloud enum fields. |
| `player_scoring_info(data)` | Prints selected identity fields for one scoring record. |
| `player_telemetry_info(data)` | Prints selected controls, settings, energy, gap, and vehicle values for one telemetry record. |
| `player_wheel_info(data)` | Prints selected values for four wheels, converting wheel index and compound type. |
| `vehicle_model_info(data, total_vehicles)` | Prints the distinct raw `mVehicleModel` byte strings among active entries. |
| `list_zero_data(data, source)` | Prints fields from `source._fields_` whose value is falsey in `data`; falsey valid values are included. |
| `test_data(info, player_index, selected_player_index)` | Coordinates the display helpers for one `SimInfo` snapshot. `player_index` is displayed; `selected_player_index` chooses the records. |
| `verify_data(info, player_index)` | Runs `list_zero_data` over session, selected scoring, selected telemetry, and the selected vehicle's front-left wheel. |

These helpers print raw byte strings and assume their supplied indices are
valid. Production code should decode strings and validate shared-memory counts
and indices first.

---

## Enum reference

All classes in `lmu_enum` inherit `enum.Enum`. The shared-memory structures
still expose raw integers (except the `c_char` caveat for yellow-flag state), so
conversion is explicit:

```python
value = lmu_enum.LMUPitState(score.mPitState)
print(value.name, value.value)
```

Direct construction raises `ValueError` for an unknown value. For a tolerant
name lookup, use `lmu_enum.enum_map(EnumClass, default="Unknown")`.

## Vehicle identity

### `LMUVehicleClass` — `LMUVehicleTelemetry.mVehicleClass`

This is the compact telemetry class code. Do not confuse it with
`LMUVehicleScoring.mVehicleClass`, which is a byte string containing a class
name.

| Value | Member | Meaning |
|---:|---|---|
| `0x00` | `Hypercar` | Hypercar class. |
| `0x02` | `LMP2_ELMS` | ELMS LMP2 class code. |
| `0x03` | `LMP2` | LMP2 class. |
| `0x04` | `LMP3` | LMP3 class. |
| `0x05` | `GTE` | GTE class. |
| `0x06` | `GT3` | GT3 class. |
| `0x08` | `PaceCar` | Pace/safety car. |
| `0xFF` | `Unknown` | Explicit unknown class sentinel. |

Values `0x01` and `0x07` have no member in this version.

### `LMUVehicleChampionship` — `LMUVehicleTelemetry.mVehicleChampionship`

| Value | Member | Meaning |
|---:|---|---|
| `0x00` | `WEC_2023` | 2023 FIA WEC content. |
| `0x01` | `WEC_2024` | 2024 FIA WEC content. |
| `0x02` | `WEC_2025` | 2025 FIA WEC content. |
| `0x03` | `WEC_2026` | 2026 FIA WEC content. |
| `0x10` | `ELMS_2025` | 2025 ELMS content. |
| `0x11` | `ELMS_2026` | 2026 ELMS content. |
| `0xFF` | `Unknown` | Explicit unknown championship sentinel. |

## Session and race state

### `LMUGameMode` — `LMUScoringInfo.mGameMode`

| Value | Member | Meaning |
|---:|---|---|
| `1` | `Server` | Game instance is acting as a server. |
| `2` | `Client` | Game instance is acting as a client. |
| `3` | `ServerAndClient` | Instance is both server and client. |

### `LMUGamePhase` — `LMUScoringInfo.mGamePhase`

The member names below are the exact names exported by the code. The game-header
descriptions in the final column are more precise for values 0 and 1.

| Value | Member | Interface meaning |
|---:|---|---|
| `0` | `Garage` | Before the session has begun. |
| `1` | `WarmUp` | Reconnaissance laps (race only). |
| `2` | `GridWalk` | Grid walk-through (race only). |
| `3` | `Formation` | Formation lap (race only). |
| `4` | `Countdown` | Starting-light countdown. |
| `5` | `GreenFlag` | Green-flag running. |
| `6` | `FullCourseYellow` | Full-course yellow/safety car. |
| `7` | `SessionStopped` | Session stopped. |
| `8` | `SessionOver` | Session over. |
| `9` | `PausedOrHeartbeat` | Paused, or a heartbeat update to the plugin. |

### `LMUYellowFlagState` — `LMUScoringInfo.mYellowFlagState`

This applies to full-course cautions. The raw structure field is
`ctypes.c_char`, so Python returns one-byte `bytes`; convert with
`int.from_bytes(raw, "little", signed=True)` before enum construction.

| Value | Member | Meaning |
|---:|---|---|
| `-1` | `Invalid` | State is invalid/unavailable. |
| `0` | `NoFlag` | No full-course yellow. |
| `1` | `Pending` | Full-course yellow pending. |
| `2` | `PitClosed` | Pits closed. |
| `3` | `PitLeadLap` | Pit lead-lap state. |
| `4` | `PitOpen` | Pits open. |
| `5` | `LastLap` | Last caution lap. |
| `6` | `Resume` | Resume state. |
| `7` | `RaceHalt` | Race halt; marked not currently used by the interface comments. |

### `LMUSession` — `LMUScoringInfo.mSession`

| Value | Member | Meaning |
|---:|---|---|
| `0` | `TestDay` | Test day. |
| `1` | `Practice1` | Practice session 1. |
| `2` | `Practice2` | Practice session 2. |
| `3` | `Practice3` | Practice session 3. |
| `4` | `Practice4` | Practice session 4. |
| `5` | `Qualifying1` | Qualifying session 1. |
| `6` | `Qualifying2` | Qualifying session 2. |
| `7` | `Qualifying3` | Qualifying session 3. |
| `8` | `Qualifying4` | Qualifying session 4. |
| `9` | `Warmup` | Warm-up session. |
| `10` | `Race1` | Race session 1. |
| `11` | `Race2` | Race session 2. |
| `12` | `Race3` | Race session 3. |
| `13` | `Race4` | Race session 4. |

### `LMUSector` — `LMUVehicleScoring.mSector`

This deliberately follows the interface's non-linear encoding.

| Value | Member | Meaning |
|---:|---|---|
| `0` | `Sector3` | Vehicle is in sector 3. |
| `1` | `Sector1` | Vehicle is in sector 1. |
| `2` | `Sector2` | Vehicle is in sector 2. |

This enum does not apply to telemetry's zero-based, pit-bit-packed
`mCurrentSector`.

### `LMUFinishStatus` — `LMUVehicleScoring.mFinishStatus`

| Value | Member | Meaning |
|---:|---|---|
| `0` | `_None` | No finish result yet. |
| `1` | `Finished` | Finished. |
| `2` | `Dnf` | Did not finish. |
| `3` | `Dq` | Disqualified. |

### `LMUControl` — `LMUVehicleScoring.mControl`

| Value | Member | Meaning |
|---:|---|---|
| `-1` | `Nobody` | Nobody controls the car; marked unexpected by the interface comments. |
| `0` | `Player` | Local player. |
| `1` | `AI` | Local AI. |
| `2` | `Remote` | Remote participant. |
| `3` | `Replay` | Replay control; marked unexpected by the interface comments. |

### `LMUPitState` — `LMUVehicleScoring.mPitState`

| Value | Member | Meaning |
|---:|---|---|
| `0` | `_None` | No pit sequence. |
| `1` | `Request` | Pit stop requested. |
| `2` | `Entering` | Entering the pits. |
| `3` | `Stopped` | Stopped in the pits. |
| `4` | `Exiting` | Exiting the pits. |

### `LMUPrimaryFlag` — `LMUVehicleScoring.mFlag`

| Value | Member | Meaning |
|---:|---|---|
| `0` | `Green` | Green/no special primary flag. |
| `6` | `Blue` | Blue flag. |

The interface comments say only these values are currently provided; future
values should be treated as possible.

### `LMUCountLapFlag` — `LMUVehicleScoring.mCountLapFlag`

| Value | Member | Meaning |
|---:|---|---|
| `0` | `DoNotCountLapOrTime` | Count neither lap nor time. |
| `1` | `CountLapButNotTime` | Count the lap but not its time. |
| `2` | `CountLapAndTime` | Count both lap and time. |

## Vehicle controls and hardware

### `LMURearFlapLegalStatus` — `LMUVehicleTelemetry.mRearFlapLegalStatus`

| Value | Member | Meaning |
|---:|---|---|
| `0` | `Disallowed` | Rear flap/DRS is disallowed. |
| `1` | `DetectedButNotAllowedYet` | Eligibility criteria detected, but activation is not allowed yet. |
| `2` | `Alllowed` | Activation allowed. The three-`l` spelling is the exact exported member name. |

### `LMUIgnitionStarterStatus` — `LMUVehicleTelemetry.mIgnitionStarter`

| Value | Member | Meaning |
|---:|---|---|
| `0` | `Off` | Ignition and starter off. |
| `1` | `Ignition` | Ignition on. |
| `2` | `IgnitionAndStarter` | Ignition and starter active. |

### `LMUWiperStatus` — `LMUVehicleTelemetry.mWiperState`

| Value | Member | Meaning |
|---:|---|---|
| `0` | `Off` | Wipers off. |
| `1` | `Auto` | Automatic wiper mode. |
| `2` | `Slow` | Slow speed. |
| `3` | `Fast` | Fast speed. |

## Wheel, tyre, and track conditions

### `LMUWheelIndex` — index into `LMUVehicleTelemetry.mWheels`

| Value | Member | Meaning |
|---:|---|---|
| `0` | `FrontLeft` | Front-left wheel. |
| `1` | `FrontRight` | Front-right wheel. |
| `2` | `RearLeft` | Rear-left wheel. |
| `3` | `RearRight` | Rear-right wheel. |

### `LMUCompoundType` — `LMUWheel.mCompoundType`

| Value | Member | Meaning |
|---:|---|---|
| `0` | `Soft` | Soft tyre category. |
| `1` | `Medium` | Medium tyre category. |
| `2` | `Hard` | Hard tyre category. |
| `3` | `Wet` | Wet-weather tyre category. |

### `LMUSurfaceType` — `LMUWheel.mSurfaceType`

| Value | Member | Meaning |
|---:|---|---|
| `0` | `Dry` | Dry racing surface. |
| `1` | `Wet` | Wet racing surface. |
| `2` | `Grass` | Grass. |
| `3` | `Dirt` | Dirt. |
| `4` | `Gravel` | Gravel. |
| `5` | `Kerb` | Rumble strip/kerb. |
| `6` | `Special` | Special surface category. |

### `LMUTrackGripLevel` — `LMUScoringInfo.mTrackGripLevel`

The value describes rubber/grip buildup, which rain can wash away.

| Value | Member | Meaning |
|---:|---|---|
| `0` | `Green` | Green/unrubbered track. |
| `1` | `Low` | Low rubber/grip. |
| `2` | `Medium` | Medium rubber/grip. |
| `3` | `High` | High/heavy rubber. |
| `4` | `Saturated` | Saturated rubber level. |

### `LMUCloudCoverage` — `LMUScoringInfo.mCloudCoverage`

| Value | Member | Meaning |
|---:|---|---|
| `0` | `Clear` | Clear sky. |
| `1` | `LightClouds` | Light clouds. |
| `2` | `PartiallyCloudy` | Partially cloudy. |
| `3` | `MostlyCloudy` | Mostly cloudy. |
| `4` | `Overcast` | Overcast. |
| `5` | `CloudyAndDrizzle` | Cloudy with drizzle. |
| `6` | `CloudyAndLightRain` | Cloudy with light rain. |
| `7` | `OvercastAndLightRain` | Overcast with light rain. |
| `8` | `OvercastAndRain` | Overcast with rain. |
| `9` | `OvercastAndHeavyRain` | Overcast with heavy rain. |
| `10` | `OvercastAndStorm` | Overcast/storm conditions. |

---

## Complete shared-memory data reference

This reference covers every field declared by the real structures in
`lmu_data.py`. It describes the repository's current layout, which its
diagnostic script identifies as the LMU 1.3 API layout.

## Type and validity conventions

| Notation | Runtime representation |
|---|---|
| `float64`, `float32` | Python `float` read from C `double` or `float`. |
| `int8/16/32`, `uint8/16/32/64` | Python `int` with the stated C storage width/sign. |
| `bool` | Python `bool` from `ctypes.c_bool`. |
| `bytes[N]` | Fixed C `char[N]`; field access returns `bytes`, normally truncated at the first NUL. |
| `uint8[N]` | Fixed `ctypes` numeric array; not `bytes` and not a tuple. |
| `Type[N]` | Fixed `ctypes` structure array. |

Every structure uses four-byte packing. Array capacities are ABI capacities,
not active element counts. Read no more than 104 vehicles, and normally no more
than `mNumVehicles` scoring or `activeVehicles` telemetry records.

Descriptions intentionally say "not specified" when the code/header comments
do not define a unit, scale, or interpretation. Reserved, unused, and pointer
storage must be treated as opaque.

## Root layout

### `LMUObjectOut` (324,820 bytes)

This is the class passed to `MMapControl` and the type of `reader.data`.

| Field | Type | Description |
|---|---|---|
| `generic` | `LMUGeneric` | Events, game version, FFB torque, and application/window state. |
| `paths` | `LMUPathData` | Five game/user filesystem paths. |
| `scoring` | `LMUScoringData` | Session-wide and per-vehicle scoring data plus results stream. |
| `telemetry` | `LMUTelemetryData` | Active vehicle selectors and per-vehicle telemetry. |

### `LMULayout` (324,820 bytes)

| Field | Type | Description |
|---|---|---|
| `data` | `LMUObjectOut` | One root data object. This wrapper has the same byte size as `LMUObjectOut`; normal readers use `LMUObjectOut` directly. |

## Generic, event, application, and path data

### `LMUGeneric` (332 bytes)

| Field | Type | Description |
|---|---|---|
| `events` | `LMUEvent` | Shared-memory event indicators. |
| `gameVersion` | `int32` | Game/interface version integer as published by LMU. Existing examples treat `0` as not running/not ready; this repository defines no formatter for nonzero values. |
| `FFBTorque` | `float32` | Current raw force-feedback torque value. Unit/range is not specified in this repository. |
| `appInfo` | `LMUApplicationState` | Game window, display, and options UI state. |

### `LMUEvent` (64 bytes)

Each field is a 32-bit event indicator. `MMapControl` only gives special
behavior to `SME_UPDATE_SCORING` and `SME_UPDATE_TELEMETRY`, testing whether
either is nonzero before refreshing a copy-mode snapshot. The repository does
not specify whether callers should interpret the other raw values as flags,
counters, or payloads beyond their associated event names.

| Field | Type | Associated event |
|---|---|---|
| `SME_ENTER` | `uint32` | Entering the plugin/interface lifecycle. |
| `SME_EXIT` | `uint32` | Exiting the plugin/interface lifecycle. |
| `SME_STARTUP` | `uint32` | Game startup. |
| `SME_SHUTDOWN` | `uint32` | Game shutdown. |
| `SME_LOAD` | `uint32` | Content/session load. |
| `SME_UNLOAD` | `uint32` | Content/session unload. |
| `SME_START_SESSION` | `uint32` | Session start. |
| `SME_END_SESSION` | `uint32` | Session end. |
| `SME_ENTER_REALTIME` | `uint32` | Transition into realtime/on-track processing. |
| `SME_EXIT_REALTIME` | `uint32` | Transition out of realtime processing. |
| `SME_UPDATE_SCORING` | `uint32` | Scoring update; used as a copy-refresh signal. |
| `SME_UPDATE_TELEMETRY` | `uint32` | Telemetry update; used as a copy-refresh signal. |
| `SME_INIT_APPLICATION` | `uint32` | Application/window state initialization. |
| `SME_UNINIT_APPLICATION` | `uint32` | Application/window state uninitialization. |
| `SME_SET_ENVIRONMENT` | `uint32` | Environment/weather update. |
| `SME_FFB` | `uint32` | Force-feedback update. |

### `LMUApplicationState` (260 bytes)

| Field | Type | Description |
|---|---|---|
| `mAppWindow` | `uint64` | Windows `HWND` value for the game window. Treat as an opaque platform handle. |
| `mWidth` | `uint32` | Screen/render width in pixels. |
| `mHeight` | `uint32` | Screen/render height in pixels. |
| `mRefreshRate` | `uint32` | Refresh rate, conventionally hertz. |
| `mWindowed` | `uint32` | Boolean-like windowed-mode value; `0` is false and nonzero is true. It is stored as `uint32`, not `bool`. |
| `mOptionsLocation` | `uint8` | UI location: `0` main UI, `1` track loading, `2` monitor, `3` on track. No enum class is provided. |
| `mOptionsPage` | `bytes[31]` | Name of the currently selected options page. |
| `mExpansion` | `uint8[204]` | Reserved for future interface expansion. Ignore. |

### `LMUPathData` (1,300 bytes)

All fields are Windows-sized path buffers and appear as `bytes`.

| Field | Type | Description |
|---|---|---|
| `userData` | `bytes[260]` | User-data directory path. |
| `customVariables` | `bytes[260]` | Custom-variables path. |
| `stewardResults` | `bytes[260]` | Steward-results path. |
| `playerProfile` | `bytes[260]` | Player-profile path. |
| `pluginsFolder` | `bytes[260]` | Plugins directory path. |

These paths can reveal local usernames and directory structure; treat raw dumps
as potentially sensitive.

## Scoring data

### `LMUScoringData` (126,832 bytes)

| Field | Type | Description |
|---|---|---|
| `scoringInfo` | `LMUScoringInfo` | Session-wide scoring, weather, server, and timing state. |
| `scoringStreamSize` | `uint8[12]` | Twelve raw bytes associated with scoring-stream size/state. Despite its name, the current Python mapping does not expose it as one numeric scalar and provides no decoder. Treat as opaque. |
| `vehScoringInfo` | `LMUVehicleScoring[104]` | Fixed-capacity per-vehicle scoring records. Only the first `scoringInfo.mNumVehicles` are normally active. |
| `scoringStream` | `bytes[65536]` | Raw scoring/results stream buffer. The interface describes results additions as newline-delimited and NUL-terminated; this library provides no parser. |

### `LMUScoringInfo` (548 bytes)

#### Session and timing

| Field | Type | Description |
|---|---|---|
| `mTrackName` | `bytes[64]` | Current track name. |
| `mSession` | `int32` | Session type code; use `LMUSession`. |
| `mCurrentET` | `float64` | Current session elapsed time, conventionally seconds. |
| `mEndET` | `float64` | Session ending elapsed time, conventionally seconds. |
| `mMaxLaps` | `int32` | Maximum configured laps. |
| `mLapDist` | `float64` | Distance around one lap, conventionally metres. |
| `mResultsStreamPointer` | `uint8[8]` | Raw in-process C pointer storage for results-stream additions. A pointer from another process is not safely dereferenceable here; use `scoringStream` instead. |
| `mNumVehicles` | `int32` | Current number of scoring vehicles; bound to `0..104` before using it as an array limit. |
| `mGamePhase` | `uint8` | Overall game phase; use `LMUGamePhase`. |
| `mYellowFlagState` | `bytes[1]` | Signed full-course yellow state stored as C `char`; convert the one-byte value before using `LMUYellowFlagState`. |
| `mSectorFlag` | `uint8[3]` | Local-yellow presence for each of three sectors. The source comments do not confirm whether element 0 is the first or last racing sector. |
| `mStartLight` | `uint8` | Current start-light animation/frame; exact number of frames depends on the track. |
| `mNumRedLights` | `uint8` | Number of red lights in the start sequence. |
| `mInRealtime` | `bool` | `True` while realtime/on track rather than at the monitor. |
| `mPlayerName` | `bytes[32]` | Local player name, including any multiplayer override. |
| `mPlrFileName` | `bytes[64]` | Player/settings filename, possibly encoded/sanitized to be a legal filename. |

#### Weather and track state

| Field | Type | Description |
|---|---|---|
| `mDarkCloud` | `float64` | Cloud darkness, documented range `0.0..1.0`. |
| `mRaining` | `float64` | Rain severity, documented range `0.0..1.0`. |
| `mAmbientTemp` | `float64` | Ambient air temperature in degrees Celsius. |
| `mTrackTemp` | `float64` | Track temperature in degrees Celsius. |
| `mWind` | `LMUVect3` | Wind velocity vector. Unit/axis convention is not stated in this repository. |
| `mMinPathWetness` | `float64` | Minimum wetness along the main path, `0.0..1.0`. |
| `mMaxPathWetness` | `float64` | Maximum wetness along the main path, `0.0..1.0`. |
| `mAvgPathWetness` | `float64` | Average wetness along the main path, `0.0..1.0`. |
| `mStartET` | `float32` | Event start time in seconds since midnight. |
| `mSessionTimeRemaining` | `float32` | Remaining session time. The structure comment does not explicitly state the unit; time fields conventionally use seconds. |
| `mTimeOfDay` | `float32` | Current time-of-day value. Unit/epoch is not documented in this repository. |
| `mIsFixedSetup` | `bool` | Whether fixed setup rules are active. |
| `mTrackGripLevel` | `uint8` | Rubber/grip level; use `LMUTrackGripLevel`. |
| `mCloudCoverage` | `uint8` | Sky/weather category; use `LMUCloudCoverage`. |
| `mTrackLimitsStepsPerPenalty` | `uint8` | Number of normalized track-limit steps corresponding to a penalty. |
| `mTrackLimitsStepsPerPoint` | `uint8` | Number of normalized steps per track-limit point. Used with telemetry `mTrackLimitsSteps`. |

#### Network/server and reserved data

| Field | Type | Description |
|---|---|---|
| `mGameMode` | `uint8` | Server/client role; use `LMUGameMode`. |
| `mIsPasswordProtected` | `bool` | Whether the server is password-protected. |
| `mServerPort` | `uint16` | Server network port when applicable. |
| `mServerPublicIP` | `uint32` | Raw numeric public IPv4 value when applicable. Byte order/string conversion is not defined here. |
| `mMaxPlayers` | `int32` | Maximum vehicles/players allowed in the session. |
| `mServerName` | `bytes[32]` | Server name. |
| `mExpansion` | `uint8[187]` | Reserved future storage. Ignore. |
| `mVehiclePointer` | `uint8[8]` | Raw in-process vehicle pointer kept at the end for ABI evolution. Do not dereference cross-process. |

### `LMUVehicleScoring` (584 bytes)

One record per scored vehicle.

#### Identity and race position

| Field | Type | Description |
|---|---|---|
| `mID` | `int32` | Slot ID used to correlate with telemetry. It can be reused after a multiplayer participant leaves. |
| `mDriverName` | `bytes[32]` | Driver name. |
| `mVehicleName` | `bytes[64]` | Vehicle name. |
| `mTotalLaps` | `int16` | Completed laps. |
| `mSector` | `int8` | Current sector using `0=sector 3`, `1=sector 1`, `2=sector 2`; use `LMUSector`. |
| `mFinishStatus` | `int8` | Finish state; use `LMUFinishStatus`. |
| `mLapDist` | `float64` | Current distance around the track. |
| `mPathLateral` | `float64` | Lateral position relative to the interface's approximate center path. |
| `mTrackEdge` | `float64` | Distance to the track edge on the same side of the center path as this vehicle. |
| `mNumPitstops` | `int16` | Pit stops completed. |
| `mNumPenalties` | `int16` | Outstanding penalties. |
| `mIsPlayer` | `bool` | Whether this is the local player's vehicle. |
| `mControl` | `int8` | Controller type; use `LMUControl`. |
| `mInPits` | `bool` | Whether the vehicle is between pit entry and pit exit. May be inaccurate for remote vehicles. |
| `mPlace` | `uint8` | Current one-based race/classification position. |
| `mVehicleClass` | `bytes[32]` | Vehicle class name string. This is distinct from telemetry's numeric class code. |

#### Lap and gap timing

All timing values in this group are conventionally seconds. Sector-2 values
explicitly marked cumulative include sector 1.

| Field | Type | Description |
|---|---|---|
| `mBestSector1` | `float64` | Best sector-1 time. |
| `mBestSector2` | `float64` | Best cumulative time through sector 2 (sector 1 + sector 2). |
| `mBestLapTime` | `float64` | Best completed lap time. |
| `mLastSector1` | `float64` | Last lap's sector-1 time. |
| `mLastSector2` | `float64` | Last lap's cumulative time through sector 2. |
| `mLastLapTime` | `float64` | Last completed lap time. |
| `mCurSector1` | `float64` | Current lap sector-1 time, when valid. |
| `mCurSector2` | `float64` | Current lap cumulative time through sector 2, when valid. |
| `mTimeBehindNext` | `float64` | Time behind the vehicle in the next higher place. |
| `mLapsBehindNext` | `int32` | Laps behind the vehicle in the next higher place. |
| `mTimeBehindLeader` | `float64` | Time behind the leader. |
| `mLapsBehindLeader` | `int32` | Laps behind the leader. |
| `mLapStartET` | `float64` | Session elapsed time when this lap started. |
| `mQualification` | `int32` | One-based qualifying position; may be `-1` when invalid. |
| `mTimeIntoLap` | `float64` | Estimated time into the current lap. |
| `mEstimatedLapTime` | `float64` | Estimated lap time used for gap and time-into-lap calculations; may vary with vehicle/setup. |
| `mBestLapSector1` | `float32` | Sector-1 component from the best lap; not necessarily the overall best sector 1. |
| `mBestLapSector2` | `float32` | Sector-2 component from the best lap; not necessarily the overall best sector 2. |

#### Motion

| Field | Type | Description |
|---|---|---|
| `mPos` | `LMUVect3` | World position in metres. |
| `mLocalVel` | `LMUVect3` | Velocity in vehicle-local coordinates, metres per second. |
| `mLocalAccel` | `LMUVect3` | Acceleration in vehicle-local coordinates, metres per second squared. |
| `mOri` | `LMUVect3[3]` | Three rows of the orientation matrix, also used to convert local coordinates. |
| `mLocalRot` | `LMUVect3` | Angular velocity in vehicle-local coordinates, radians per second. |
| `mLocalRotAccel` | `LMUVect3` | Angular acceleration in local coordinates, radians per second squared. |

#### Pit, flags, vehicle metadata, and reserved data

| Field | Type | Description |
|---|---|---|
| `mHeadlights` | `uint8` | Headlight status as a raw integer/boolean-like value. |
| `mPitState` | `uint8` | Pit sequence state; use `LMUPitState`. |
| `mServerScored` | `uint8` | Boolean-like value indicating whether the server is scoring this vehicle; it may be off in some qualifying/race heats. |
| `mIndividualPhase` | `uint8` | Per-vehicle game phase. In addition to general game phases, comments define `9=after formation`, `10=under yellow`, `11=under blue (unused)`. No dedicated enum covers those extensions. |
| `mPitGroup` | `bytes[24]` | Pit group, normally the team name unless a pit is shared. |
| `mFlag` | `uint8` | Primary flag shown to the vehicle; use `LMUPrimaryFlag` for currently documented values. |
| `mUnderYellow` | `bool` | Whether the car has taken a full-course caution flag at start/finish. |
| `mCountLapFlag` | `uint8` | Whether to count the lap and/or time; use `LMUCountLapFlag`. |
| `mInGarageStall` | `bool` | Whether the car appears to be in its correct garage stall. |
| `mUpgradePack` | `uint8[16]` | Encoded vehicle upgrades. This repository provides no decoder. |
| `mPitLapDist` | `float32` | Pit location expressed as lap distance. |
| `mSteamID` | `uint64` | Current driver's Steam ID, if any. Potentially sensitive identifier. |
| `mVehFilename` | `bytes[32]` | `.veh` filename used to identify the vehicle. |
| `mAttackMode` | `int16` | Raw attack-mode value. Meaning/range is not documented in this repository. |
| `mFuelFraction` | `uint8` | Fuel or battery remaining encoded from `0x00` (0%) to `0xFF` (100%). Convert with `value / 255`. |
| `mDRSState` | `bool` | DRS/rear-flap state. Exact distinction between availability and activation is not further documented here. |
| `mExpansion` | `uint8[4]` | Reserved future storage. Ignore. |

## Telemetry data

### `LMUTelemetryData` (196,356 bytes)

| Field | Type | Description |
|---|---|---|
| `activeVehicles` | `uint8` | Number of active telemetry records; bound to `0..104` before using as an array limit. |
| `playerVehicleIdx` | `uint8` | Array index of the local player's telemetry record. Validate it before indexing. |
| `playerHasVehicle` | `bool` | Whether a local player vehicle is currently available. |
| `telemInfo` | `LMUVehicleTelemetry[104]` | Fixed-capacity telemetry records. Normally only the first `activeVehicles` entries are active. |

### `LMUVehicleTelemetry` (1,888 bytes)

One record per active telemetry vehicle.

#### Identity, time, and motion

| Field | Type | Description |
|---|---|---|
| `mID` | `int32` | Slot ID used to correlate with scoring. Can be reused after a multiplayer participant leaves. |
| `mDeltaTime` | `float64` | Time since this record's last telemetry update, seconds. |
| `mElapsedTime` | `float64` | Game/session elapsed time, seconds. |
| `mLapNumber` | `int32` | Current lap number. |
| `mLapStartET` | `float64` | Session elapsed time when the current lap began. |
| `mVehicleName` | `bytes[64]` | Current vehicle name. |
| `mTrackName` | `bytes[64]` | Current track name. |
| `mPos` | `LMUVect3` | World position in metres. |
| `mLocalVel` | `LMUVect3` | Velocity in local vehicle coordinates, metres per second. |
| `mLocalAccel` | `LMUVect3` | Acceleration in local coordinates, metres per second squared. |
| `mOri` | `LMUVect3[3]` | Three rows of the orientation matrix, also used for local-coordinate conversion. |
| `mLocalRot` | `LMUVect3` | Angular velocity in local coordinates, radians per second. |
| `mLocalRotAccel` | `LMUVect3` | Angular acceleration in local coordinates, radians per second squared. |

#### Driver inputs, transmission, and engine

| Field | Type | Description |
|---|---|---|
| `mGear` | `int32` | Gear: `-1` reverse, `0` neutral, `1+` forward gears. |
| `mEngineRPM` | `float64` | Current engine speed in revolutions per minute. |
| `mEngineWaterTemp` | `float64` | Engine coolant/water temperature in degrees Celsius. |
| `mEngineOilTemp` | `float64` | Engine oil temperature in degrees Celsius. |
| `mClutchRPM` | `float64` | Clutch speed in revolutions per minute. |
| `mUnfilteredThrottle` | `float64` | Raw throttle input, `0.0..1.0`. |
| `mUnfilteredBrake` | `float64` | Raw brake input, `0.0..1.0`. |
| `mUnfilteredSteering` | `float64` | Raw steering input, `-1.0..1.0`, left to right. |
| `mUnfilteredClutch` | `float64` | Raw clutch input, `0.0..1.0`. |
| `mFilteredThrottle` | `float64` | Game-filtered throttle, `0.0..1.0`. |
| `mFilteredBrake` | `float64` | Game-filtered brake, `0.0..1.0`. |
| `mFilteredSteering` | `float64` | Game-filtered steering, `-1.0..1.0`, left to right. |
| `mFilteredClutch` | `float64` | Game-filtered clutch, `0.0..1.0`. |
| `mSteeringShaftTorque` | `float64` | Torque about the steering shaft. Unit/range is not stated here. |
| `mEngineMaxRPM` | `float64` | Engine rev limit in RPM. |
| `mEngineTorque` | `float64` | Current engine torque including additive torque. Unit is not stated here. |
| `mMaxGears` | `uint8` | Maximum number of forward gears. |
| `mIgnitionStarter` | `uint8` | Ignition/starter state; use `LMUIgnitionStarterStatus`. |
| `mAntiStallActivated` | `uint8` | Boolean-like value for hard anti-stall activation. |

#### Chassis, aero, fuel, damage, and controls

| Field | Type | Description |
|---|---|---|
| `mFront3rdDeflection` | `float64` | Front third-spring deflection. Unit is not specified here. |
| `mRear3rdDeflection` | `float64` | Rear third-spring deflection. Unit is not specified here. |
| `mFrontWingHeight` | `float64` | Front wing height. Unit/reference is not specified here. |
| `mFrontRideHeight` | `float64` | Front ride height. Unit/reference is not specified here. |
| `mRearRideHeight` | `float64` | Rear ride height. Unit/reference is not specified here. |
| `mDrag` | `float64` | Current aerodynamic drag value. Unit is not specified here. |
| `mFrontDownforce` | `float64` | Current front downforce value. Unit is not specified here. |
| `mRearDownforce` | `float64` | Current rear downforce value. Unit is not specified here. |
| `mFuel` | `float64` | Fuel amount in litres. |
| `mScheduledStops` | `uint8` | Number of scheduled pit stops. |
| `mOverheating` | `bool` | Whether the overheating icon is displayed. |
| `mDetached` | `bool` | Whether any non-wheel parts have detached. Wheel detachment is reported per wheel. |
| `mHeadlights` | `bool` | Whether headlights are on. |
| `mDentSeverity` | `uint8[8]` | Dent severity at eight positions around the car: `0` none, `1` some, `2` more. The position-to-index map is not supplied here. |
| `mLastImpactET` | `float64` | Session elapsed time of the last impact. |
| `mLastImpactMagnitude` | `float64` | Magnitude of the last impact; unit/scale is not specified. |
| `mLastImpactPos` | `LMUVect3` | World location of the last impact. |
| `mCurrentSector` | `int32` | Zero-based current sector with pit-lane state stored in the high/sign bit. Interpret as a 32-bit bit pattern. |
| `mSpeedLimiter` | `uint8` | Boolean-like legacy value indicating the speed limiter is on. |
| `mFuelCapacity` | `float64` | Fuel tank capacity in litres. |
| `mFrontFlapActivated` | `uint8` | Boolean-like front flap activation. |
| `mRearFlapActivated` | `uint8` | Boolean-like rear flap activation. |
| `mRearFlapLegalStatus` | `uint8` | Rear flap/DRS eligibility; use `LMURearFlapLegalStatus`. |
| `mSpeedLimiterAvailable` | `uint8` | Boolean-like value indicating a limiter is available. |
| `mVisualSteeringWheelRange` | `float32` | Visual steering-wheel range. Unit is not specified here. |
| `mRearBrakeBias` | `float64` | Fraction of braking assigned to the rear. Multiply by 100 for percentage. |
| `mTurboBoostPressure` | `float64` | Current turbo boost pressure when available. Unit is not specified here. |
| `mPhysicsToGraphicsOffset` | `float32[3]` | Three-component offset from static center of gravity to graphical center. |
| `mPhysicalSteeringWheelRange` | `float32` | Physical steering-wheel range. Unit is not specified here. |
| `mDeltaBest` | `float64` | Raw delta-to-best value. Unit and sign convention are not documented here. |
| `mLapInvalidated` | `bool` | Whether the current lap is invalidated. |
| `mABSActive` | `bool` | Whether ABS is actively intervening. |
| `mTCActive` | `bool` | Whether traction control is actively intervening. |
| `mSpeedLimiterActive` | `bool` | Whether the speed limiter is actively limiting. |
| `mWiperState` | `uint8` | Wiper setting; use `LMUWiperStatus`. |

#### Tyre selection

| Field | Type | Description |
|---|---|---|
| `mFrontTireCompoundIndex` | `uint8` | Front compound index within the tyre brand. |
| `mRearTireCompoundIndex` | `uint8` | Rear compound index within the tyre brand. |
| `mFrontTireCompoundName` | `bytes[18]` | Front tyre compound name. |
| `mRearTireCompoundName` | `bytes[18]` | Rear tyre compound name. |

#### Hybrid/electric system

| Field | Type | Description |
|---|---|---|
| `mBatteryChargeFraction` | `float64` | Battery charge as a fraction, `0.0..1.0`. |
| `mElectricBoostMotorTorque` | `float64` | Boost motor torque; can be negative during regeneration. Unit is not stated here. |
| `mElectricBoostMotorRPM` | `float64` | Boost motor speed in RPM. |
| `mElectricBoostMotorTemperature` | `float64` | Boost motor temperature; unit is not specified here. |
| `mElectricBoostWaterTemperature` | `float64` | Boost motor cooler water temperature, or `0` when unavailable; unit is not specified here. |
| `mElectricBoostMotorState` | `uint8` | `0` unavailable, `1` inactive, `2` propulsion, `3` regeneration. No enum class is provided. |
| `mRegen` | `float32` | Regeneration power in kW. |
| `mStateOfCharge` | `float32` | Battery state of charge in percent (unlike `mBatteryChargeFraction`). |
| `mVirtualEnergy` | `float32` | Virtual energy as a fraction. |

#### Onboard settings and progress

The comments identify these as onboard settings and their maximum adjustable
steps. The meaning of individual numeric steps depends on the vehicle.

| Field | Type | Description |
|---|---|---|
| `mTC` | `uint8` | Current traction-control setting step. |
| `mTCMax` | `uint8` | Maximum traction-control setting step. |
| `mTCSlip` | `uint8` | Current traction-control slip setting. |
| `mTCSlipMax` | `uint8` | Maximum traction-control slip setting. |
| `mTCCut` | `uint8` | Current traction-control cut setting. |
| `mTCCutMax` | `uint8` | Maximum traction-control cut setting. |
| `mABS` | `uint8` | Current ABS setting. |
| `mABSMax` | `uint8` | Maximum ABS setting. |
| `mMotorMap` | `uint8` | Current motor/engine map setting. |
| `mMotorMapMax` | `uint8` | Maximum motor-map setting. |
| `mMigration` | `uint8` | Current brake/energy migration setting. Exact semantics are vehicle-dependent and not defined here. |
| `mMigrationMax` | `uint8` | Maximum migration setting. |
| `mFrontAntiSway` | `uint8` | Current front anti-roll/anti-sway setting. |
| `mFrontAntiSwayMax` | `uint8` | Maximum front anti-sway setting. |
| `mRearAntiSway` | `uint8` | Current rear anti-roll/anti-sway setting. |
| `mRearAntiSwayMax` | `uint8` | Maximum rear anti-sway setting. |
| `mLiftAndCoastProgress` | `uint8` | Raw lift-and-coast progress value. Scale/range is not documented here. |
| `mTrackLimitsSteps` | `uint8` | Normalized track-limit total: track-limit points multiplied by session `mTrackLimitsStepsPerPoint`. |

#### Gaps, vehicle classification, wheels, and reserved fields

| Field | Type | Description |
|---|---|---|
| `mTimeGapCarAhead` | `float32` | Time gap to the car physically ahead. Unit/sentinel behavior is not documented here. |
| `mTimeGapCarBehind` | `float32` | Time gap to the car physically behind. |
| `mTimeGapPlaceAhead` | `float32` | Time gap to the vehicle in the next higher position. |
| `mTimeGapPlaceBehind` | `float32` | Time gap to the vehicle in the next lower position. |
| `mVehicleModel` | `bytes[30]` | Vehicle brand and model name. |
| `mVehicleClass` | `uint8` | Compact vehicle class code; use `LMUVehicleClass`. Distinct from scoring's class-name bytes. |
| `mVehicleChampionship` | `uint8` | Championship/year code; use `LMUVehicleChampionship`. |
| `mUnused` | `uint8[2]` | Unused padding/storage. Ignore. |
| `mExpansion` | `uint8[20]` | Reserved future storage. Ignore. |
| `mWheels` | `LMUWheel[4]` | Wheel records in front-left, front-right, rear-left, rear-right order; use `LMUWheelIndex`. |

### `LMUWheel` (260 bytes)

One record for each wheel. Wheel-local left/right temperature positions are
physical left/center/right, not tyre inside/center/outside.

#### Suspension, motion, and forces

| Field | Type | Description |
|---|---|---|
| `mSuspensionDeflection` | `float64` | Suspension deflection in metres. |
| `mRideHeight` | `float64` | Wheel-area ride height in metres. |
| `mSuspForce` | `float64` | Pushrod/suspension load in newtons. |
| `mBrakeTemp` | `float64` | Brake temperature in degrees Celsius. |
| `mBrakePressure` | `float64` | Currently a normalized `0.0..1.0` value based on driver input and brake balance. The interface comment warns it may become true kPa pressure in the future. |
| `mRotation` | `float64` | Wheel angular speed in radians per second. |
| `mLateralPatchVel` | `float64` | Lateral contact-patch velocity. Unit is not explicitly stated; velocity fields conventionally use m/s. |
| `mLongitudinalPatchVel` | `float64` | Longitudinal contact-patch velocity. |
| `mLateralGroundVel` | `float64` | Lateral ground velocity at the contact patch. |
| `mLongitudinalGroundVel` | `float64` | Longitudinal ground velocity at the contact patch. |
| `mCamber` | `float64` | Camber in radians. Positive points left for left-side wheels and right for right-side wheels. |
| `mLateralForce` | `float64` | Lateral tyre force in newtons. |
| `mLongitudinalForce` | `float64` | Longitudinal tyre force in newtons. |
| `mTireLoad` | `float64` | Tyre load in newtons. |
| `mGripFract` | `float64` | Approximation of the fraction of the contact patch that is sliding. |

#### Tyre state and geometry

| Field | Type | Description |
|---|---|---|
| `mPressure` | `float64` | Tyre pressure in kPa. |
| `mTemperature` | `float64[3]` | Surface temperature in Kelvin at physical left, center, right. Subtract `273.15` for Celsius. |
| `mWear` | `float64` | Wear fraction `0.0..1.0` of maximum wear. It is not necessarily proportional to grip loss and is not “grip remaining.” |
| `mTerrainName` | `bytes[16]` | Contact material prefix from the game's TDF file. |
| `mSurfaceType` | `uint8` | Surface category; use `LMUSurfaceType`. |
| `mFlat` | `bool` | Whether the tyre is flat. |
| `mDetached` | `bool` | Whether the wheel is detached. |
| `mStaticUndeflectedRadius` | `uint8` | Static undeflected tyre radius in centimetres. |
| `mVerticalTireDeflection` | `float64` | Deflection from the tyre's speed-sensitive radius. Unit is not explicitly stated. |
| `mWheelYLocation` | `float64` | Wheel Y position relative to the vehicle Y position. Unit/axis direction is not explicitly stated. |
| `mToe` | `float64` | Current toe angle relative to the vehicle. Unit/sign convention is not stated. |
| `mTireCarcassTemperature` | `float64` | Rough average carcass temperature in Kelvin. |
| `mTireInnerLayerTemperature` | `float64[3]` | Rough average innermost-rubber-layer temperature in Kelvin at left, center, right. |
| `mOptimalTemp` | `float32` | Optimal tyre temperature in degrees Celsius. Unlike the measured tyre temperature arrays, this is already Celsius. |
| `mCompoundIndex` | `uint8` | Index into the compounds available for this car/track. |
| `mCompoundType` | `uint8` | Broad compound category; use `LMUCompoundType`. |
| `mExpansion` | `uint8[18]` | Reserved future storage. Ignore. |

## Vector type

### `LMUVect3` (24 bytes)

| Field | Type | Description |
|---|---|---|
| `x` | `float64` | X component. Meaning/unit comes from the parent field. |
| `y` | `float64` | Y component. Meaning/unit comes from the parent field. |
| `z` | `float64` | Z component. Meaning/unit comes from the parent field. |

The repository does not declare a universal handedness or axis-direction
convention. Do not label one component as forward/up without validating it
against LMU's interface header or observed data for the targeted game version.

## Structure size checklist

The manual diagnostic asserts these exact ABI sizes:

| Structure | Bytes |
|---|---:|
| `LMUVect3` | 24 |
| `LMUWheel` | 260 |
| `LMUVehicleTelemetry` | 1,888 |
| `LMUVehicleScoring` | 584 |
| `LMUScoringInfo` | 548 |
| `LMUApplicationState` | 260 |
| `LMUScoringData` | 126,832 |
| `LMUTelemetryData` | 196,356 |
| `LMUPathData` | 1,300 |
| `LMUEvent` | 64 |
| `LMUGeneric` | 332 |
| `LMUObjectOut` | 324,820 |
| `LMULayout` | 324,820 |

If a game update changes the native headers, a size match alone is necessary
but not sufficient: field order, types, packing, array counts, and semantics
must all remain aligned.
