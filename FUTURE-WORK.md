# dxlAPRS Fork — Changes & Future Work

This fork (`YogySK/dxlAPRS`) adds extended JSON output fields for the YogyTracker ecosystem.

## Completed Changes

### Extended JSON Output (sondeaprs.c/sondeaprs.h)
- **EXTDATA struct**: Added `fstate`, `bk`, `tmphum`, `ozonExtV`, `mesok`, `ozonInstType`, `ozonInstNum`, `imetTi`, `imetTp`, `imetTu`, `dewp`
- **wrcsv JSON**: All EXTDATA fields emitted in appropriate nested objects (`ptu{}`, `aux{}`, top-level)
- **All decoder reset blocks** updated with new field initialization

### RS41 (existing support, enhanced)
- `fstate` (flight state: 0=ground, 1=flight, 2=descent)
- `bk` (burst-kill armed)
- `tmphum` → `ptu.th` (humidity sensor temperature)
- `cal` (calibration percent)
- `hrms`/`vrms` (GPS RMS errors)

### RS92 (sondemod.c)
- `mesok` → top-level `"mesok":0|1` (sensor heating status)

### iMET (sondemod.c)
- Ozone instrument ID: `aux.otyp`, `aux.onum` (uncommented from existing code)
- Extended PTU: `ptu.ti`, `ptu.tp`, `ptu.tu` (type 0x04 handler added)

### C34/C50 (sondemod.c)
- Dew point: `ptu.dewp` (case '\007' handler added)

### M20 (sondemod.c)
- **Temperature**: NTC thermistor decoding at bytes 0x04-0x05 (with header offset 16)
  - Same Steinhart-Hart coefficients as M10 (EPCOS B57540G0502)
  - Scale/range extracted from upper bits of 16-bit ADC value
- **Battery voltage**: byte 0x26, val × 3.3/255
- GPS satellite count confirmed NOT available in M20 frame

### DFM (sondemod.c)
- **Temperature**: NTC thermistor decoding from CFG block measurement channels
  - CONTEXTDFM6 extended with `meas24[9]`, `cfgchk24[9]`, `raw16[9]`, `sn_ch`
  - fl24 custom float format decoder (4-bit exponent + 20-bit mantissa)
  - B-parameter Steinhart-Hart: T = 1/(p0 + p1·ln(R) + p2·ln(R)² + p3·ln(R)³) - 273.15
  - Supports DFM-06 (Rf=220kΩ), DFM-09 (Rf=220kΩ), DFM-17 (Rf=332kΩ)
  - P-sensor variant detection (DFM-09P, DFM-17P: shifted measurement indices)
- **Battery voltage**: raw16 at conf_id 5 (or 7 for P-type) / 1000.0 V (DFM-09+)

### sdrtst Noise Metric Output (sdrtest.c)
- `showrssi()` now appends `:medmed` (smoothed FM noise level) after the dB/db suffix
- Output changed from `67.0dB 1` to `67.0dB:0.80 1`
- `medmed` is an exponential moving average of `sqsum` (per-block FM noise variance), α=0.1
- Enables YogyFeeder to display accurate squelch threshold bar: threshold = `squelchPct × 0.16` (same as sdrtst's `lev = sq*32/200`)

### YogyFeeder Pipeline (separate repo)
- `satdb[]` per-satellite signal levels: parsed, forwarded to YogyTracker
- `xdata[]` raw auxiliary frames: parsed (xdata, xdata1, xdata2), forwarded
- All new fields: heatingOk, dewPoint, imetExtPtu, ozone instrumentType/Number

## Future Work

### DFM Humidity
DFM radiosondes have humidity sensors but do NOT transmit calibration coefficients. The factory calibration is stored on EEPROM and only available to GRAW ground station software. No open-source decoder can decode DFM humidity from the radio telemetry alone.

### M20 GPS Satellites
Confirmed not present in the M20 frame structure. Unlike M10 which has satellite count at byte 0x1E, the M20 frame does not include this field.

### MEISEI PTU Enhancement
The MEISEI/IMS100 decoder may have additional PTU fields that could be extracted. Low priority.
