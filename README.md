# WitMotion WT901 — artificial horizon card and BLE integration

Two related pieces:

- `custom_components/witmotion_level/` — a local BLE integration that connects to a
  WitMotion WT901BLE IMU and publishes its angles, acceleration, G-force and braking.
- `www/community/wt901-horizon-card/wt901-horizon-card.js` — a Lovelace card
  (`custom:wt901-horizon-card`, v0.1.0) that renders an artificial horizon from those sensors.

The card is not usable without the integration. Neither piece is in HACS.

## Hardware and BLE identity

The module advertises as e.g. `WT901BLE67` (prefix `WT901BLE`). Its BLE address can change
between sessions, so the integration matches on the advertised service UUID rather than on a
fixed MAC address. Only one WT901 entry is supported (unique ID `witmotion_level`).

GATT layout, verified against the device on 2026-11-09:

| Role | UUID |
|---|---|
| Service | `0000ffe5-0000-1000-8000-00805f9a34fb` |
| Notify (data stream) | `0000ffe4-0000-1000-8000-00805f9a34fb` |
| Write | `0000ffe9-0000-1000-8000-00805f9a34fb` (declared; the integration never writes to it) |

## Installation

1. Copy `custom_components/witmotion_level/` into `/config/custom_components/` and restart HA.
2. With the module powered on and in range it normally appears under
   Settings → Devices & Services → Discovered. Otherwise add it manually with
   “Add Integration → WitMotion WT901 Neigungssensor”.
3. Copy `wt901-horizon-card.js` into `/config/www/community/wt901-horizon-card/`.
4. In Lovelace, add a resource: URL `/local/community/wt901-horizon-card/wt901-horizon-card.js`,
   type `JavaScript module`. After editing the JS, bump the resource `?v=` — browsers cache
   `/local/` resources for about 31 days.
5. Add a card of type `custom:wt901-horizon-card`.

## Entities

Device: “WT901 Neigungssensor” (manufacturer WitMotion, model WT901BLE).

| Key | Name | Unit |
|---|---|---|
| `angle_x` | Winkel X | ° |
| `angle_y` | Winkel Y | ° |
| `angle_z` | Winkel Z | ° |
| `accel_x` / `accel_y` / `accel_z` | Beschleunigung X / Y / Z | g |
| `g_force` | Gesamt-G | g |
| `brake` | Bremsverzögerung | m/s² |

Entity IDs in this installation:

```
sensor.wt901_neigungssensor_winkel_x
sensor.wt901_neigungssensor_winkel_y
sensor.wt901_neigungssensor_winkel_z
sensor.wt901_neigungssensor_beschleunigung_x
sensor.wt901_neigungssensor_beschleunigung_y
sensor.wt901_neigungssensor_beschleunigung_z
sensor.wt901_neigungssensor_gesamt_g
sensor.wt901_neigungssensor_bremsverzogerung
```

## Protocol and derived values

The module streams `0x55`-tagged frames. Both frame shapes are accepted: 11-byte frames
(`0x51` accel, `0x52` gyro, `0x53` angle, `0x54` magnetometer, `0x50` time) and 20-byte
combined frames (`0x61`, `0x62`). Unknown tags are skipped byte by byte and the buffer is
capped, so a corrupt stream cannot grow it without bound.

Scaling is int16 full scale: 32768 counts = 16 g, 180° or 2000°/s.

- `g_force` is the magnitude of the acceleration vector (≈1.0 at rest).
- `brake` is `max(0, -(accel_x − baseline) × 9.80665 − 0.05)` m/s². The baseline tracks the
  signal with a ~5 s time constant so a fixed mounting tilt does not read as permanent
  braking; dynamic deceleration is much faster than the baseline and survives the subtraction.
  The forward axis is X and the sign assumes a confirmed mounting orientation.
- Entity updates are throttled to at most 2 Hz even though the module can stream near 100 Hz,
  to avoid flooding the recorder.

## Connection behaviour

An active Bluetooth callback watches for the service UUID; each new advertisement from a
disconnected module triggers a connect using the latest advertised address. Connection uses
`bleak_retry_connector` with up to 3 attempts (connecting this module took ~9 s in practice).
A watchdog reconnects after 90 s without notifications, and a supervisor loop retries every
15 s if advertisements stop arriving.

## Card configuration

| Option | Default | Meaning |
|---|---|---|
| `pitch_entity` | `sensor.wt901_neigungssensor_winkel_y` | Pitch source (Winkel **Y**) |
| `roll_entity` | `sensor.wt901_neigungssensor_winkel_x` | Roll source (Winkel **X**) |
| `g_entity` | `sensor.wt901_neigungssensor_gesamt_g` | G-force readout |
| `brake_entity` | `sensor.wt901_neigungssensor_bremsverzogerung` | Braking readout |
| `title` | `""` | Optional heading; HTML is escaped |
| `invert_pitch` | `false` | Negate pitch |
| `invert_roll` | `false` | Negate roll |
| `pitch_offset` / `roll_offset` | `0` | Degrees added before inversion |
| `px_per_degree` | `4.2` | Horizon scale; ladder marks at ±10/20/30° |
| `show_readouts` | `true` | Show PITCH / ROLL / G-KRAFT numeric readouts |
| `show_brake` | `true` | Show the BREMSEN readout (only when `show_readouts` is true) |
| `smooth_alpha` | `0.35` | Exponential smoothing factor, clamped to 0.05–0.95 |

Note the axis mapping: pitch comes from **Y** and roll from **X**. Offsets are applied before
inversion. The card requires a configuration object and throws if none is supplied; it
reports a card size of 7 rows.

Example:

```yaml
type: custom:wt901-horizon-card
title: Niveau
pitch_entity: sensor.wt901_neigungssensor_winkel_y
roll_entity: sensor.wt901_neigungssensor_winkel_x
g_entity: sensor.wt901_neigungssensor_gesamt_g
brake_entity: sensor.wt901_neigungssensor_bremsverzogerung
px_per_degree: 4.2
smooth_alpha: 0.35
```

If the module is mounted at a different orientation, add e.g.:

```yaml
invert_pitch: true
invert_roll: true
pitch_offset: -1.2
roll_offset: 0.8
```
