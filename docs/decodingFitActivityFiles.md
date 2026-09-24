# Decoding FIT Activity Files with fitdecode

This document covers how to use the `fitdecode` Python library to parse `.fit` activity files, access message data, and convert encoded values to usable units.

---

## Installation

```bash
pip install fitdecode
```

---

## Core Concepts

`fitdecode` reads a FIT file as a stream of **frames**. Each frame is one of four types:

| Frame Type Constant       | Class                 | When emitted                                  |
|---------------------------|-----------------------|-----------------------------------------------|
| `FIT_FRAME_HEADER`        | `FitHeader`           | At the start of each FIT file in the stream   |
| `FIT_FRAME_DEFINITION`    | `FitDefinitionMessage`| When a message schema is declared             |
| `FIT_FRAME_DATA`          | `FitDataMessage`      | For each data record (session, record, etc.)  |
| `FIT_FRAME_CRC`           | `FitCRC`              | At end of each FIT file                       |

For activity parsing, only `FIT_FRAME_DATA` frames are needed.

---

## FitReader

### Constructor

```python
fitdecode.FitReader(
    fileish,                          # path string, Path, or binary file-like object
    processor=DefaultDataProcessor(), # applies scale/offset and converts timestamps
    check_crc=CrcCheck.WARN,          # CRC validation mode
    error_handling=ErrorHandling.WARN # how to handle malformed records
)
```

**Key parameters:**

- `fileish` — accepts a file path or any object opened in binary mode (`rb`). Pass `io.BytesIO(bytes_data)` to parse from memory.
- `processor=None` — disables all automatic value decoding (scale, offset, timestamps); raw integers are returned.
- `check_crc=CrcCheck.RAISE` — raises `FitCRCError` on checksum mismatch.
- `error_handling=ErrorHandling.RAISE` — raises `FitParseError` on any malformed record.

### Basic iteration

```python
import fitdecode

with fitdecode.FitReader('activity.fit') as fit:
    for frame in fit:
        if frame.frame_type == fitdecode.FIT_FRAME_DATA:
            print(frame.name, [f.name for f in frame.fields])
```

### Parsing from bytes (in-memory)

```python
import io
import fitdecode

def parse_bytes(file_bytes: bytes):
    with fitdecode.FitReader(io.BytesIO(file_bytes)) as fit:
        for frame in fit:
            ...
```

---

## FitDataMessage

The object yielded for every data record.

### Attributes

| Attribute          | Type                    | Description                                   |
|--------------------|-------------------------|-----------------------------------------------|
| `frame_type`       | int (constant 4)        | Always `FIT_FRAME_DATA`                       |
| `name`             | str                     | Message name (e.g., `"record"`, `"session"`)  |
| `global_mesg_num`  | int                     | FIT global message number                     |
| `fields`           | list[FieldData]         | All fields present in this message            |
| `def_mesg`         | FitDefinitionMessage    | The definition that describes this message    |
| `is_developer_data`| bool                    | True for messages with developer-defined fields|

### Methods

#### `has_field(field_name_or_num) → bool`

Returns `True` if the field is present in this message.

```python
if frame.has_field('heart_rate'):
    hr = frame.get_value('heart_rate')
```

#### `get_value(field_name_or_num, *, fallback=..., raw_value=False) → Any`

Returns the decoded field value. Raises `KeyError` if the field is absent and no `fallback` is provided.

```python
hr      = frame.get_value('heart_rate', fallback=None)
speed   = frame.get_value('speed', fallback=None)           # m/s (scale already applied)
raw_alt = frame.get_value('altitude', raw_value=True)       # raw integer before scale/offset
```

#### `get_field(field_name_or_num, idx=0) → FieldData`

Returns the `FieldData` object for a field. Useful when you also need units or metadata.

```python
field = frame.get_field('speed')
print(field.value, field.units)   # e.g., 3.5, 'm/s'
```

#### `get_fields(field_name_or_num) → Generator[FieldData]`

Yields all `FieldData` objects matching the name (for messages with duplicate field identifiers).

---

## FieldData

Returned by `get_field()` and contained in `frame.fields`.

| Attribute     | Type       | Description                               |
|---------------|------------|-------------------------------------------|
| `name`        | str        | Field name (e.g., `"heart_rate"`)         |
| `value`       | Any        | Decoded value (scale/offset applied)      |
| `raw_value`   | Any        | Raw integer before decoding               |
| `units`       | str | None | Unit string (e.g., `"bpm"`, `"m/s"`)     |
| `def_num`     | int        | Field definition number in the FIT profile|

---

## Reading an Activity File

### Pattern: collect session and records

```python
import io
import fitdecode

SEMICIRCLES_TO_DEGREES = 180.0 / (2 ** 31)

def parse_activity(file_bytes: bytes) -> dict:
    session_data = {}
    records = []

    with fitdecode.FitReader(io.BytesIO(file_bytes)) as fit:
        for frame in fit:
            if frame.frame_type != fitdecode.FIT_FRAME_DATA:
                continue

            if frame.name == 'session':
                session_data = _extract_session(frame)

            elif frame.name == 'record':
                records.append(_extract_record(frame))

    return {'session': session_data, 'records': records}


def _extract_session(frame: fitdecode.FitDataMessage) -> dict:
    def val(name):
        return frame.get_value(name, fallback=None)

    return {
        'sport':              val('sport'),
        'start_time':         val('start_time'),
        'total_elapsed_time': val('total_elapsed_time'),   # seconds (float)
        'total_distance':     val('total_distance'),       # meters (float)
        'total_calories':     val('total_calories'),
        'avg_speed':          val('enhanced_avg_speed') or val('avg_speed'),   # m/s
        'max_speed':          val('enhanced_max_speed') or val('max_speed'),
        'avg_heart_rate':     val('avg_heart_rate'),
        'max_heart_rate':     val('max_heart_rate'),
        'avg_cadence':        val('avg_cadence'),
        'avg_power':          val('avg_power'),
        'total_ascent':       val('total_ascent'),
        'total_descent':      val('total_descent'),
        'avg_temperature':    val('avg_temperature'),
    }


def _extract_record(frame: fitdecode.FitDataMessage) -> dict:
    def val(name):
        return frame.get_value(name, fallback=None)

    lat = val('position_lat')
    lon = val('position_long')

    return {
        'timestamp':   val('timestamp'),
        'heart_rate':  val('heart_rate'),
        'cadence':     val('cadence'),
        'speed':       val('enhanced_speed') or val('speed'),     # m/s
        'power':       val('power'),
        'temperature': val('temperature'),
        'altitude':    val('enhanced_altitude') or val('altitude'),  # meters
        'distance':    val('distance'),                              # cumulative meters
        'position_lat': lat * SEMICIRCLES_TO_DEGREES if lat is not None else None,
        'position_long': lon * SEMICIRCLES_TO_DEGREES if lon is not None else None,
    }
```

---

## GPS Coordinate Conversion

`fitdecode` does **not** automatically convert semicircles to degrees. The raw `sint32` value must be converted manually:

```python
SEMICIRCLES_TO_DEGREES = 180.0 / (2 ** 31)   # ≈ 8.381903e-8

lat_raw = frame.get_value('position_lat', fallback=None)
lon_raw = frame.get_value('position_long', fallback=None)

if lat_raw is not None and lon_raw is not None:
    lat_deg = lat_raw * SEMICIRCLES_TO_DEGREES
    lon_deg = lon_raw * SEMICIRCLES_TO_DEGREES
```

GPS fields are absent for indoor activities — always guard with `if lat_raw is not None`.

---

## Speed and Altitude Conversion

`fitdecode` applies scale and offset automatically, so values are already in real units:

```python
# speed is already in m/s — convert to km/h for display:
speed_ms  = frame.get_value('speed', fallback=None)       # m/s
speed_kph = speed_ms * 3.6 if speed_ms is not None else None

# altitude is already in meters:
altitude_m = frame.get_value('altitude', fallback=None)    # m
```

---

## Timestamps

With `DefaultDataProcessor` (the default), `timestamp` fields are returned as Python `datetime.datetime` objects in UTC. No manual conversion is needed:

```python
ts = frame.get_value('timestamp')   # datetime.datetime (UTC)
```

---

## Exception Handling

| Exception               | When raised                                                    |
|-------------------------|----------------------------------------------------------------|
| `fitdecode.FitParseError` | File is not a valid FIT file, or a record is malformed       |
| `fitdecode.FitCRCError`   | Checksum validation failed (subclass of `FitParseError`)     |
| `KeyError`              | `get_value()` called for a field not present (use `fallback`) |

### Wrapping parse errors

```python
import io
import fitdecode

class UnparseableFileError(Exception):
    pass

def safe_parse(file_bytes: bytes):
    try:
        with fitdecode.FitReader(io.BytesIO(file_bytes)) as fit:
            for frame in fit:
                ...
    except fitdecode.FitParseError as exc:
        raise UnparseableFileError(f"Invalid FIT data: {exc}") from exc
```

---

## Message Names Reference

Common message names encountered when iterating a `FitDataMessage`:

| `frame.name`   | Purpose                                         |
|----------------|-------------------------------------------------|
| `file_id`      | File metadata (device, serial, creation time)   |
| `session`      | Aggregate activity summary                      |
| `lap`          | Per-lap summary                                 |
| `record`       | Per-sample telemetry                            |
| `event`        | Start/stop/pause markers                        |
| `device_info`  | Paired sensors and device metadata              |
| `activity`     | Links sessions; provides local timestamp        |
| `sport`        | Sport/sub-sport declaration                     |

---

## Common Pitfalls

**Prefer `enhanced_*` fields** — Newer devices write `enhanced_altitude` and `enhanced_speed` instead of `altitude` and `speed`. Always try enhanced first:

```python
alt = frame.get_value('enhanced_altitude', fallback=None) \
      or frame.get_value('altitude', fallback=None)
```

**GPS is not always present** — Indoor rides, treadmill runs, and pool swims will have no `position_lat`/`position_long` fields. Guard every GPS read.

**Cadence semantics differ by sport** — For running, many devices record half-cadence (steps on one foot per minute). Double the value if the sport is running.

**`None` vs absent** — A field can be in the definition but have a value of `None` (invalid/missing sentinel). `has_field()` returning `True` does not guarantee the value is non-`None`.

**`fallback` prevents `KeyError`** — Always use `fallback=None` when a field may be optional:

```python
hr = frame.get_value('heart_rate', fallback=None)
```

---

## References

- [fitdecode on PyPI](https://pypi.org/project/fitdecode/)
- [fitdecode GitHub](https://github.com/polyvertex/fitdecode)
- [fitdecode API docs](https://fitdecode.readthedocs.io/en/latest/)
- [fitdecode records reference](https://fitdecode.readthedocs.io/en/latest/reference/records.html)
- [Garmin FIT SDK](https://developer.garmin.com/fit/overview/)
