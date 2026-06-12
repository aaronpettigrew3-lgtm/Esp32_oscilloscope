# ESP32 Oscilloscope — ADC Calibration Plan (revised)

## Executive summary

**Current state:** the firmware reads raw ADC counts with `adc1_get_raw()` (12-bit,
11 dB attenuation) and ships **raw 0–4095 counts** to the browser. There is *no*
voltage conversion in the firmware at all — the JS canvas just scales counts to pixels
([oscilloscope.h:1461](oscilloscope.h#L1461),
[amber_oscilloscope_html.h:880](amber/amber_oscilloscope_html.h#L880)).

**Goal:** add factory (eFuse) calibration so counts can be shown as real millivolts,
without slowing the sample loop and without meaningful RAM cost.

**Decision (locked in):** stay on the **legacy `esp_adc_cal` / `driver/adc.h`** API to
match the existing `adc1_get_raw` code path. Smallest diff; emits a deprecation warning
on Arduino-ESP32 core 3.x but compiles and works.

**Realistic accuracy:** raw ±10% → **±1–3%** across the calibrated range with eFuse
calibration; **±0.5%** only with an additional *user* two-point calibration against a
precision reference. The original "±10% → ±0.5% from two changes" was optimistic.

---

## Corrections to the previous plan (important)

These were wrong or outdated in the first draft and are not part of the approach:

1. **`esp_adc_cal.h` is deprecated** on ESP-IDF v5 / core 3.x. We keep it deliberately
   (consistent with the legacy `adc1_get_raw` path), not by accident.
2. **There is no hardware averaging on the classic ESP32 oneshot ADC.** The
   `adc_digi_config_t{ .samples = 16 }` snippet was fabricated — that struct/field does
   not exist. Averaging is **software only**.
3. **No runtime temperature compensation exists in the library.** `esp_adc_cal`
   characterizes once; it never adjusts for temperature. Dropped.
4. **No curve-fit linearization on classic ESP32.** Only line fitting is available
   (curve fitting is S2/S3/C3 only). Any polynomial would be hand-rolled. Dropped.
5. **No 8 KB LUT needed.** Because classic-ESP32 calibration is linear, a slope+intercept
   pair (~8 bytes) reproduces it exactly. This is the memory-friendly core of the plan.

---

## Hardware limits that cap accuracy regardless of software

- **11 dB attenuation is only linear ~0.15 V–2.45 V.** Below ~0.1 V and above ~2.5 V the
  ADC saturates/clips. Near 0 V and near 3.3 V cannot be measured accurately by any
  software means. (Newer cores rename `ADC_ATTEN_DB_11` → `ADC_ATTEN_DB_12`.)
- **Source impedance must be low (< ~10 kΩ)** and a **100 nF cap** on the ADC pin is
  Espressif's explicit noise recommendation. Do this before chasing software percent.
- **ADC2 is unusable here** — this is a WiFi-AP scope and ADC2 is blocked while WiFi is
  active ([oscilloscope.h:1167](oscilloscope.h#L1167)). Use ADC1 only (GPIO 32–39).
- **Many pre-2018 chips have no eFuse calibration.** They fall back to the default
  1100 mV Vref and gain almost nothing. The firmware must report which scheme it got so
  the user isn't misled.

---

## Approach (memory-aware, sample-loop-safe)

### 1. Characterize once at init
After the existing attenuation config
([oscilloscope.h:1461](oscilloscope.h#L1461)):

```cpp
#include <esp_adc_cal.h>

esp_adc_cal_characteristics_t adc_chars;
esp_adc_cal_value_t cal_type = esp_adc_cal_characterize(
    ADC_UNIT_1, ADC_ATTEN_DB_11, ADC_WIDTH_BIT_12, 1100, &adc_chars);
// cal_type ∈ { TP, EFUSE_VREF, DEFAULT_VREF } — report this to the user.
```

### 2. Derive a linear count→mV mapping (no LUT, ~8 bytes)
Classic-ESP32 calibration is linear, so two evaluations fully describe it:

```cpp
uint32_t mv_lo = esp_adc_cal_raw_to_voltage(0,    &adc_chars);  // intercept
uint32_t mv_hi = esp_adc_cal_raw_to_voltage(4095, &adc_chars);  // for slope
float mv_per_count = (float)(mv_hi - mv_lo) / 4095.0f;
// store mv_lo (offset) and mv_per_count (slope) — that's the whole calibration
```

This avoids any per-sample library call **and** any lookup table. The hot reader loop
([oscilloscope.h:248-274](oscilloscope.h#L248)) keeps calling `adc1_get_raw` unchanged.

### 3. Convert at the edge, not in the loop
Ship the two coefficients (`offset`, `slope`) and `cal_type` to the browser **once**, and
compute `mV = raw * slope + offset` in JS at render time for axis labels / cursor
readouts. Zero firmware hot-path cost, zero extra RAM. (Alternative: apply the MAC in
firmware only when serializing a frame — also cheap — if we'd rather not touch the JS.)

### 4. Averaging only in a future DC mode
Multisampling divides effective sample rate by N and low-pass-filters the signal — wrong
for live waveforms. If a dedicated slow "voltmeter" mode is added later, use 16–64×
software averaging there only. Not part of the fast timebases.

---

## Implementation steps

1. `#include <esp_adc_cal.h>` and add the characterization + coefficient derivation right
   after the `adc1_config_channel_atten` block ([oscilloscope.h:1462](oscilloscope.h#L1462)).
2. Store `offset`, `slope`, `cal_type` in the scope's shared/config struct.
3. Emit cal status to the dmesg/serial log (and ideally to the JS client) so the user sees
   `TwoPoint` / `eFuse Vref` / `Default Vref`.
4. Wire the two coefficients into the websocket/config handshake and add a voltage axis
   (or cursor mV readout) in the HTML/JS.
5. (Optional, later) user two-point calibration vs. a known reference, stored in NVS, to
   reach ±0.5%.

**RAM cost:** ~12 bytes (two floats + an enum). **Flash:** the `esp_adc_cal` code, already
in the core. **Hot-loop cost:** none (raw read unchanged).

---

## Testing

1. **Accuracy:** feed 0.5 V, 1.0 V, 1.65 V, 2.0 V from a bench supply (stay inside the
   0.15–2.45 V linear window) and compare to a DMM.
2. **Cal scheme:** confirm the logged `cal_type` — if `Default Vref`, expect little gain.
3. **Stability:** hold a steady voltage 60 s, check spread (and the effect of a 100 nF cap).
4. **No regression:** verify sample rate / waveform timing is unchanged after the edits.

---

## Sources
- ADC calibration driver (current): https://docs.espressif.com/projects/esp-idf/en/stable/esp32/api-reference/peripherals/adc/adc_calibration.html
- Legacy ADC (eFuse Vref / two-point, noise/multisampling): https://docs.espressif.com/projects/esp-idf/en/v4.4/esp32/api-reference/peripherals/adc.html
- v5 migration (legacy deprecation): https://docs.espressif.com/projects/esp-idf/en/release-v5.0/esp32c2/migration-guides/release-5.x/peripherals.html
- Two-point calibration scheme: https://docs.espressif.com/projects/esp-iot-solution/en/latest/others/adc_tp_calibration.html
