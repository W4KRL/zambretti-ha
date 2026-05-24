# Zambretti Weather Forecast for Home Assistant

An enhanced implementation of the Zambretti (1920) barometric weather
forecasting algorithm for Home Assistant, using triggered template sensors
and a Markdown dashboard card.

Copyright (c) 2026 Karl W. Berger — MIT License  
Developed with AI assistance from Claude (Anthropic).

---

## Background

The Zambretti Forecaster was devised by Negretti and Zambra in 1920 as a
mechanical slide-rule weather forecaster based on barometric pressure and
trend. It produces one of 26 forecast codes (A through Z) based on the
current sea-level pressure and whether pressure is rising, steady, or
falling over a 3-hour observation window. Despite its age, the algorithm
claims approximately 90% accuracy for 12-hour forecasts in temperate
maritime climates and remains one of the most effective simple forecasting
methods available to the amateur meteorologist.

---

## Files

| File | Description |
|---|---|
| `zambretti.yaml` | Original Zambretti — 3-level trend (rising/steady/falling) |
| `zambretti_enhanced.yaml` | Enhanced Zambretti — 5-level NOAA trend, snow substitution |
| `dashboards/zambretti_card.yaml` | Markdown dashboard card for original |
| `dashboards/zambretti_enhanced_card.yaml` | Markdown dashboard card for enhanced |

Both sensor files may be active simultaneously. Dashboard cards call
independent sensor entities suffixed `_enh` for the enhanced version.

---

## Requirements

### Statistics Sensor
Add to your `configuration.yaml`:

```yaml
sensor:
  - platform: statistics
    name: "wx3 pressure stats"
    entity_id: YOUR_PRESSURE_SENSOR_ENTITY_ID
    state_characteristic: change
    max_age:
      hours: 3
    sampling_size: 6480
    precision: 2
```

Replace `YOUR_PRESSURE_SENSOR_ENTITY_ID` with your actual pressure sensor
entity ID. The `change` characteristic returns the net pressure change over
the 3-hour window — this is the delta value driving all trend calculations.

### Sensor Entity IDs
Both yaml files reference these entities — replace with your own:

| Entity | Description |
|---|---|
| `sensor.wx3_pressure_stats` | Statistics sensor output (change over 3h) |
| `sensor.wx3_weather_station_2ad0b0_wx_pressure` | Current sea-level pressure in mbar |
| `sensor.wx3_weather_station_2ad0b0_wx_temperature` | Current temperature in °F (enhanced only) |

---

## Installation

1. Copy `zambretti.yaml` and/or `zambretti_enhanced.yaml` to your HA
   config directory or packages folder.

2. Reference them in `configuration.yaml`. If using a packages folder:

```yaml
homeassistant:
  packages: !include_dir_named YOUR_FOLDER
```

3. Add the statistics sensor shown above.

4. Reload Template Entities: **Developer Tools → YAML → Template Entities**.

5. Add a Markdown card to your dashboard and paste the contents of the
   appropriate card file from the `dashboards/` folder.

---

## Sensor Architecture

Each yaml file produces four sensors evaluated simultaneously on a
5-minute clock trigger:

```
sensor.wx3_pressure_stats ──► Sensor #2  Barometer Trend Index
                          ──► Sensor #3  Barometer Trend Indication
                          ──► Sensor #4  Zambretti Forecast
sensor.wx3_..._pressure   ──► Sensor #4  Zambretti Forecast
sensor.wx3_..._temperature──► Sensor #4  Zambretti Forecast (enhanced only)

Sensor #1  SLP 3h Delta   ── display/logging only, not in computation chain
```

Sensors #2, #3, and #4 read `sensor.wx3_pressure_stats` directly rather
than chaining through Sensor #1. This eliminates timing errors that would
occur if downstream sensors read a value committed in a previous trigger
cycle. Sensor #1 exists solely as a display convenience for dashboards.

The dashboard cards likewise read `sensor.wx3_pressure_stats` directly
for the delta and trend indication display, ensuring all displayed values
are consistent at the moment of rendering.

---

## The 26 Zambretti Codes

All forecast output uses the original 26 Zambretti codes verbatim. No new
codes are invented in either implementation.

### Wording Notes

**Code A** is rendered as `A: Settled Fine` rather than the common
`A: Settled Weather`. In 1920s British meteorological vernacular "settled"
meant stable fine weather, not merely stable conditions. The two-word form
`Settled Fine` captures both the stability and the fair weather implied by
the highest pressure bands across all three trend tables.

**Codes H, I, G, N, O** use `Showers` rather than Zambretti's original
`Showery`. This is a deliberate transliteration of 1920s British
meteorological vernacular into modern plain English — "showery" described
shower frequency and likelihood, not precipitation type. These codes are
therefore **not** subject to snow substitution in the enhanced version.

### Falling Table

| Pressure (mbar) | Code | Forecast |
|---|---|---|
| > 1045 | A | Settled Fine |
| > 1032 | B | Fine Weather |
| > 1020 | D | Fine, Becoming Less Settled |
| > 1014 | H | Fairly Fine, Showers Later |
| > 1006 | O | Showers, Becoming More Unsettled |
| > 1000 | R | Unsettled, Rain Later |
| > 993  | U | Rain At Times, Worse Later |
| > 987  | V | Rain At Times, Becoming Very Unsettled |
| ≤ 987  | X | Very Unsettled, Rain |

### Steady Table

| Pressure (mbar) | Code | Forecast |
|---|---|---|
| > 1028 | A | Settled Fine |
| > 1017 | B | Fine Weather |
| > 1011 | E | Fine, Possible Showers |
| > 1003 | K | Fairly Fine, Showers Likely |
| > 996  | N | Showers, Bright Intervals |
| > 991  | P | Changeable, Some Rain |
| > 984  | S | Unsettled, Rain At Times |
| > 978  | W | Rain At Frequent Intervals |
| > 966  | X | Very Unsettled, Rain |
| ≤ 966  | Z | Stormy, Much Rain |

### Rising Table

| Pressure (mbar) | Code | Forecast |
|---|---|---|
| > 1025 | A | Settled Fine |
| > 1016 | B | Fine Weather |
| > 1009 | C | Becoming Fine |
| > 1003 | F | Fairly Fine, Improving |
| > 997  | G | Fairly Fine, Possible Showers Early |
| > 992  | I | Showers Early, Improving |
| > 986  | J | Changeable, Mending |
| > 980  | L | Rather Unsettled, Clearing Later |
| > 973  | M | Unsettled, Probably Improving |
| > 967  | Q | Unsettled, Short Fine Intervals |
| > 961  | T | Very Unsettled, Finer At Times |
| > 953  | Y | Stormy, Possibly Improving |
| ≤ 953  | Z | Stormy, Much Rain |

---

## Enhancements (zambretti_enhanced.yaml)

### 1. Five-Level NOAA Trend

The original Zambretti uses three trend levels: rising, steady, falling.
The NOAA standard defines five: Rising Rapidly, Rising Slowly, Steady,
Falling Slowly, Falling Rapidly. The thresholds used are:

| Delta (mbar/3h) | Trend |
|---|---|
| > 2.0 | Rising Rapidly |
| 0.5 to 2.0 | Rising Slowly |
| −0.5 to 0.5 | Steady |
| −2.0 to −0.5 | Falling Slowly |
| < −2.0 | Falling Rapidly |

### 2. Next-Better / Next-Worse Letter Selection

For rapid trend cases the same pressure bands as the slow trend tables are
used, but each band selects the adjacent Zambretti letter on the severity
scale rather than inventing new codes:

**Falling Rapidly** — each band selects one step worse than Falling Slowly:
```
A→B  B→D  D→H  H→O  O→R  R→U  U→V  V→X  X→Z
```

**Rising Rapidly** — each band selects one step better than Rising Slowly:
```
Z→Y  Y→T  T→Q  Q→M  M→L  L→J  J→I  I→G  G→F  F→C  C→B  B→A
```

The top two Rising Rapidly bands both resolve to A — the best available
code. No new forecast codes are invented. Every output string is one of
the original 26 Zambretti codes.

### 3. Snow Substitution

When temperature is at or below 35°F, precipitation forecasts substitute
Snow for Rain. Applied only to codes that explicitly contain the word
"Rain": R, S, U, V, W, X, Z, P. Codes using "Showers" (G, H, I, N, O)
are unaffected — see wording notes above.

---

## Dashboard Card

The Markdown card computes trend indication inline from
`sensor.wx3_pressure_stats` rather than reading
`sensor.barometer_trend_indication` or `sensor.barometer_trend_indication_enh`.
This ensures the displayed trend is always consistent with the current
delta value at render time, regardless of the sensor trigger cycle.

Color coding:

| Color | Condition |
|---|---|
| Blue `#0078D7` | Settled / Fine |
| Green `#00A300` | Improving / Bright Intervals |
| Orange `#FFA500` | Showers / Changeable / Unsettled |
| Red `#D83B01` | Rain / Snow / Stormy |

---

## Planned Enhancements

- WMO Code Table 4680 pressure tendency shape detection to handle
  non-monotonic pressure behavior (oscillating pressure within the
  3-hour window)
- Three-panel diagnostic visualization: absolute pressure, 3-hour
  delta, and WMO tendency code overlay

---

## Reference

Negretti and Zambra, *A Handbook of Meteorological Instruments*, 1920.  
WMO Code Table 4680 — Characteristic of pressure tendency.  
NOAA Barometric Pressure Tendency definitions.
