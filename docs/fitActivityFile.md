# FIT Activity File Reference

FIT (Flexible and Interoperable Data Transfer) is a compact binary format developed by Garmin for sports, fitness, and health device data. This document covers the file structure and message definitions relevant to parsing activity files.

---

## File Structure

A FIT file is composed of three sections:

```
[ File Header ] [ Records... ] [ File CRC (2 bytes) ]
```

### File Header (14 bytes)

| Offset | Size | Field            | Notes                                           |
|--------|------|------------------|-------------------------------------------------|
| 0      | 1    | Header Size      | Always 14 (or 12 for legacy files)              |
| 1      | 1    | Protocol Version | e.g., 0x10 = protocol 1.0                       |
| 2      | 2    | Profile Version  | e.g., 2132 = profile 21.32                      |
| 4      | 4    | Data Size        | Byte count of all records (excludes header/CRC) |
| 8      | 4    | Data Type        | ASCII `.FIT` (0x2E 0x46 0x49 0x54)              |
| 12     | 2    | CRC              | Header CRC; 0x0000 if unused                    |

### Record Header (1 byte per record)

Every record begins with a header byte that describes what follows:

```
Bit 7: Header type — 0 = Normal, 1 = Compressed Timestamp
Bit 6: Message type — 0 = Data Message, 1 = Definition Message  (Normal header only)
Bit 5: Developer data flag                                        (Normal header only)
Bit 4: Reserved
Bits 3-0: Local Message Type (0–15) — links data to its definition
```

### Definition Messages

A definition message declares the schema for subsequent data messages of the same local message type:

| Field              | Size     | Notes                                    |
|--------------------|----------|------------------------------------------|
| Reserved           | 1 byte   | Always 0                                 |
| Architecture       | 1 byte   | 0 = little-endian, 1 = big-endian        |
| Global Message Num | 2 bytes  | Which FIT message type (e.g., 20=record) |
| Num Fields         | 1 byte   | How many field definitions follow        |
| Field Definitions  | 3×N bytes| (field_def_num, size_bytes, base_type)   |

### Data Messages

Data messages carry the actual values, ordered and typed per their associated definition message. The parser maps local message types back to global ones to decode them.

---

## Message Types for Activity Files

Activity files contain four required message types, plus optional ones:

| Global Num | Name       | Required | Description                                    |
|------------|------------|----------|------------------------------------------------|
| 0          | `file_id`  | Yes      | Must be the first message; identifies the file |
| 18         | `session`  | Yes      | Aggregate totals for the whole activity        |
| 19         | `lap`      | Optional | Split summary per manual or auto lap           |
| 20         | `record`   | Yes      | Per-sample telemetry (usually ~1 Hz)           |
| 21         | `event`    | Optional | Start/stop/pause markers                       |
| 34         | `activity` | Yes      | Links sessions; contains local timestamp       |

---

## Field Reference

### `file_id` Message (global_mesg_num = 0)

| Field           | Type     | Units | Notes                         |
|-----------------|----------|-------|-------------------------------|
| `type`          | enum     | —     | 4 = activity file             |
| `manufacturer`  | uint16   | —     | Manufacturer ID               |
| `product`       | uint16   | —     | Product/device ID             |
| `serial_number` | uint32z  | —     | Device serial number          |
| `time_created`  | datetime | —     | File creation timestamp       |

### `session` Message (global_mesg_num = 18)

One session message per activity (multisport activities may have one per sport leg).

| Field                    | Type     | Scale | Offset | Units  | Notes                               |
|--------------------------|----------|-------|--------|--------|-------------------------------------|
| `timestamp`              | datetime | —     | —      | —      | End-of-session timestamp            |
| `start_time`             | datetime | —     | —      | —      | Activity start                      |
| `start_position_lat`     | sint32   | —     | —      | semicircles | GPS latitude at start          |
| `start_position_long`    | sint32   | —     | —      | semicircles | GPS longitude at start         |
| `sport`                  | enum     | —     | —      | —      | running, cycling, swimming, etc.    |
| `sub_sport`              | enum     | —     | —      | —      | e.g., road, trail, indoor           |
| `total_elapsed_time`     | uint32   | 1000  | —      | s      | Wall-clock duration                 |
| `total_timer_time`       | uint32   | 1000  | —      | s      | Active (paused time excluded)       |
| `total_distance`         | uint32   | 100   | —      | m      | Total distance covered              |
| `total_calories`         | uint16   | —     | —      | kcal   |                                     |
| `avg_speed`              | uint16   | 1000  | —      | m/s    | Use `enhanced_avg_speed` if absent  |
| `max_speed`              | uint16   | 1000  | —      | m/s    |                                     |
| `enhanced_avg_speed`     | uint32   | 1000  | —      | m/s    | Higher resolution                   |
| `enhanced_max_speed`     | uint32   | 1000  | —      | m/s    |                                     |
| `avg_heart_rate`         | uint8    | —     | —      | bpm    |                                     |
| `max_heart_rate`         | uint8    | —     | —      | bpm    |                                     |
| `avg_cadence`            | uint8    | —     | —      | rpm    | Cycling: rpm; Running: steps/min    |
| `max_cadence`            | uint8    | —     | —      | rpm    |                                     |
| `avg_power`              | uint16   | —     | —      | W      |                                     |
| `max_power`              | uint16   | —     | —      | W      |                                     |
| `total_ascent`           | uint16   | —     | —      | m      |                                     |
| `total_descent`          | uint16   | —     | —      | m      |                                     |
| `avg_temperature`        | sint8    | —     | —      | °C     |                                     |
| `max_temperature`        | sint8    | —     | —      | °C     |                                     |
| `enhanced_avg_altitude`  | uint32   | 5     | 500    | m      | Decoded: `value/5 - 500`            |
| `enhanced_max_altitude`  | uint32   | 5     | 500    | m      |                                     |
| `enhanced_min_altitude`  | uint32   | 5     | 500    | m      |                                     |

### `record` Message (global_mesg_num = 20)

One message per sample, typically captured every 1–5 seconds depending on the device.

| Field               | Type   | Scale | Offset | Units       | Notes                                       |
|---------------------|--------|-------|--------|-------------|---------------------------------------------|
| `timestamp`         | datetime | —   | —      | —           | Sample timestamp                            |
| `position_lat`      | sint32 | —     | —      | semicircles | Convert to degrees (see below)              |
| `position_long`     | sint32 | —     | —      | semicircles | Convert to degrees (see below)              |
| `altitude`          | uint16 | 5     | 500    | m           | Decoded: `value/5 - 500`                    |
| `enhanced_altitude` | uint32 | 5     | 500    | m           | Higher precision; prefer over `altitude`    |
| `heart_rate`        | uint8  | —     | —      | bpm         |                                             |
| `cadence`           | uint8  | —     | —      | rpm         |                                             |
| `distance`          | uint32 | 100   | —      | m           | Cumulative distance                         |
| `speed`             | uint16 | 1000  | —      | m/s         |                                             |
| `enhanced_speed`    | uint32 | 1000  | —      | m/s         | Higher precision; prefer over `speed`       |
| `power`             | uint16 | —     | —      | W           |                                             |
| `temperature`       | sint8  | —     | —      | °C          |                                             |
| `grade`             | sint16 | 100   | —      | %           | Road incline                                |
| `accumulated_power` | uint32 | —     | —      | W           | Running total of power output               |

### `lap` Message (global_mesg_num = 19)

Contains the same aggregated fields as `session` but scoped to a single lap interval. Key additional fields:

| Field             | Type     | Notes                      |
|-------------------|----------|----------------------------|
| `message_index`   | uint16   | Zero-based lap index       |
| `start_time`      | datetime | Lap start                  |
| `end_position_lat`| sint32   | GPS at lap end (semicircles) |
| `end_position_long`| sint32  | GPS at lap end (semicircles) |
| `lap_trigger`     | enum     | manual, distance, time, etc. |

---

## Data Encoding

### GPS Coordinates — Semicircles

Latitude and longitude are stored as 32-bit signed integers in **semicircles**:

```
degrees = semicircles × (180 / 2³¹)
degrees = semicircles × (180 / 2,147,483,648)
degrees = semicircles × 8.381903171539307e-8
```

Example:
```python
SEMICIRCLES_TO_DEGREES = 180.0 / (2 ** 31)

lat_deg = record.get_value('position_lat') * SEMICIRCLES_TO_DEGREES
lon_deg = record.get_value('position_long') * SEMICIRCLES_TO_DEGREES
```

### Timestamps — FIT Epoch

FIT timestamps count **seconds since 1989-12-31 00:00:00 UTC** (not the Unix epoch). `fitdecode` with `DefaultDataProcessor` converts them automatically to Python `datetime` objects in UTC.

### Scale and Offset

Many integer fields encode decimal values via scale/offset:

```
real_value = (raw_integer / scale) - offset
```

`fitdecode` with `DefaultDataProcessor` applies scale and offset automatically — `get_value()` returns the already-decoded real value (e.g., speed in m/s, altitude in m). Raw integers can be retrieved with `get_raw_value()`.

**Exception**: GPS semicircles are NOT converted automatically — you must apply the formula above.

---

## Field Availability

FIT files vary significantly by manufacturer, device model, and sport type:

- **GPS fields** (`position_lat`, `position_long`) are absent for indoor activities and treadmill runs.
- **Power** is only present when a power meter or smart trainer is connected.
- **Temperature** requires a compatible sensor or built-in thermometer.
- **Cadence** semantics differ by sport: cycling = pedal RPM, running = steps per minute (half-cadence on some devices).
- **`enhanced_*` fields** (altitude, speed) provide higher precision and should be preferred when available.
- Some devices write only a `session` summary with no `record` messages.

---

## Sport Type Enum (common values)

| Value | Name        |
|-------|-------------|
| 0     | generic     |
| 1     | running     |
| 2     | cycling     |
| 5     | swimming    |
| 10    | hiking      |
| 15    | walking     |
| 255   | all         |

---

## References

- [Garmin FIT SDK Overview](https://developer.garmin.com/fit/overview/)
- [FIT File Types](https://developer.garmin.com/fit/file-types/)
- [FIT Protocol Reference](https://developer.garmin.com/fit/protocol/)
- [Suunto FIT Field Descriptions](https://apizone.suunto.com/fit-description)
