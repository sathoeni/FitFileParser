# Profile.xlsx Modifications

**SDK Version**: 21.171

## Summary

This document tracks custom modifications made to `Profile.xlsx` beyond the standard FIT SDK distribution. These modifications are necessary to enable parsing of sensor data messages that would otherwise cause crashes or be inaccessible.

## Problem Fixed

Without these modifications, the library would crash with `EXC_BAD_ACCESS` when encountering accelerometer, gyroscope, or magnetometer data in FIT files. The crash occurred because the code generator (`fitsdkparser.py`) only creates C struct definitions for messages when their fields have example values in the Profile.xlsx "Example" column.

## Modified Messages

Added example values (column P) to enable parsing for these sensor messages:

### Message 164: gyroscope_data
Fields with examples added:
- timestamp: `1`
- timestamp_ms: `1`
- sample_time_offset: `1000`
- gyro_x: `1`
- gyro_y: `1`
- gyro_z: `1`
- calibrated_gyro_x: `1`
- calibrated_gyro_y: `1`
- calibrated_gyro_z: `1`

### Message 165: accelerometer_data
Fields with examples added:
- timestamp: `1`
- timestamp_ms: `1`
- sample_time_offset: `1000`
- accel_x: `1`
- accel_y: `1`
- accel_z: `1`
- calibrated_accel_x: `1`
- calibrated_accel_y: `1`
- calibrated_accel_z: `1`

### Message 208: magnetometer_data
Fields with examples added:
- timestamp: `1`
- timestamp_ms: `1`
- sample_time_offset: `1000`
- mag_x: `1`
- mag_y: `1`
- mag_z: `1`
- calibrated_mag_x: `1`
- calibrated_mag_y: `1`
- calibrated_mag_z: `1`

### Message 209: barometer_data
Fields with examples added:
- timestamp: `1`
- timestamp_ms: `1`
- sample_time_offset: `1000`
- baro_pres: `1`

### Message 376: hsa_gyroscope_data (High-Sampling Gyroscope)
Fields with examples added:
- timestamp: `1`
- timestamp_ms: `1`
- sample_time_offset: `1000` (Note: This is an array field, value indicates array size)
- gyro_x: `1`
- gyro_y: `1`
- gyro_z: `1`

## When Updating FIT SDK

When updating to a new FIT SDK version, follow these steps to preserve the modifications:

1. **Before running `fitsdkupdate.py`:**
   ```bash
   cp python/Profile.xlsx python/Profile.xlsx.backup
   ```

2. **Run the SDK update:**
   ```bash
   ./python/fitsdkupdate.py /path/to/new/FitSDKRelease_XX.XXX.XX
   ```

3. **Re-apply the modifications:**
   - Open the new `python/Profile.xlsx` in Excel or LibreOffice Calc
   - Navigate to the "Messages" tab
   - For each message listed above, add the example values in column P (Example)
   - Alternatively, use the Python script at `python/apply_sensor_modifications.py` (if created)

4. **Regenerate the code:**
   ```bash
   cd python
   ./fitsdkparser.py generate Profile.xlsx
   ```

5. **Build and test:**
   ```bash
   swift build
   swift test
   ```

## Additional Code Changes

### rzfit_swift_map.swift

Added Swift constant for FLOAT32 invalid value (line ~10):
```swift
// FLOAT32 invalid constant (C macro not bridged to Swift)
let FIT_FLOAT32_INVALID: Float = .nan
```

This constant is required because:
- Calibrated sensor fields use FLOAT32 type
- The C macro `FIT_FLOAT32_INVALID` is not automatically bridged to Swift
- The generated code needs this constant to check for invalid values

**Note**: This addition is made to the auto-generated file and will need to be re-added if the file is regenerated. Consider modifying `fitsdkparser.py` to automatically include this constant.

## Verification

To verify the modifications are working:

1. **Check C struct definitions exist:**
   ```bash
   grep "FIT_ACCELEROMETER_DATA_MESG" Sources/FitFileParserObjc/rzfit_objc_reference_mesg.h
   ```
   Should return a line showing the struct definition.

2. **Check Swift case statements exist:**
   ```bash
   grep "case 165:" Sources/FitFileParser/rzfit_swift_map.swift | head -1
   ```
   Should show: `case 165: // accelerometer_data`

3. **Test with sensor data file:**
   ```swift
   let fit = FitFile(file: urlToFitFileWithSensorData, parsingType: .generic)
   let accelMessages = fit.messages(forMessageType: 165)
   // Should not crash and should return messages
   ```

## Impact

- **Crash fix**: Prevents `EXC_BAD_ACCESS` when parsing files with sensor data
- **Fast mode**: Enables fast parsing mode for sensor messages (2-3x faster than generic mode)
- **Data access**: Makes sensor data accessible through standard API
- **Compatibility**: Both `.fast` and `.generic` parsing modes now work with sensor data

## References

- Original issue: `EXC_BAD_ACCESS (code=1, address=0xa900006e0100ac)` at `fit_convert.m:348`
- Fix date: 2025-12-12
- FIT SDK version: 21.171
- Code generator: `python/fitsdkparser.py`
