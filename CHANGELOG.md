# Changelog

## 0.1.1 — 2026-11-12

Moves off its stand-ins onto devices-nv 0.4.0 and embedded-hal-nv 0.4.0. The release depends on both.

- `Accelerometer.accel_rate_mhz` and `accel_set_rate_mhz` state the rates in
  thousandths of a hertz, so the low-power modes' 1.6 Hz and the 12.5 Hz that a
  whole-hertz request rounds up from are reachable through the trait. The
  module gains `odr_for_rate_mhz` and `rate_write_mhz`; `accel_rate_hz` and
  `accel_set_rate_hz` are unchanged.
- `Accelerometer.accel_mode` and `accel_set_mode` take `AccelMode`:
  `AccelLowPower` is Low-Power Mode 1 with the low-noise setting off,
  `AccelNormal` is High-Performance Mode with it off, and `AccelHighResolution`
  is High-Performance Mode with it on. `accel_mode` reads the single-conversion
  mode as `AccelLowPower`. The inherent `mode`, `lp_level`, `resolution` and
  `set_mode` stay.
- `Accelerometer.accel_start` starts one conversion on demand and answers its
  length in microseconds, or -1: the part is armed for the single-conversion
  mode when it is not in it, and stays in that mode after the conversion. The
  inherent `convert` stays, for a part that is armed already.
- `Thermometer.temp_offset_error_milli_c` answers 15000, the offset of Table 5.

## 0.1.0

The first release: a driver for ST's LIS2DW12 three-axis accelerometer on an
I2C or an SPI bus.

- `Lis2dw12` (I2C) and `Lis2dw12Spi` (SPI), the types a board's device table
  builds, implementing devices-nv's `Accelerometer`, including the FIFO,
  `Thermometer` and `Identifiable`, with their own methods for the mode, the
  low-power level, the width, the FIFO, the conversion on demand and the
  self-test.
- The register access as functions over `Lis2dw12Port`, a trait that a type
  holding a bus implements, and the framing of each bus as functions over
  `I2cBus` and `SpiBus`.
- Readings in milli-g at the sensitivity of Table 3, 14 bits in
  High-Performance Mode and in Low-Power Modes 2 to 4 and 12 bits in
  Low-Power Mode 1, converted with the settings read in the same transaction
  as the sample.
- The rates of Table 29 in both families of modes, the low-noise setting and
  the filter bandwidth.
- The die temperature from OUT_T_L and OUT_T_H.
- The FIFO as an option: off after `begin`, on in continuous mode through
  `fifo_begin`.
- The conversion on demand with the software trigger, and two functions for a
  host's scheduler that park the task for it (`accel_measure`) or for the
  data-ready flag (`accel_next`).  `irq_arm` and `irq_disarm` arm the INT1
  line through an `AsyncEdge` source.
- The wake-up, activity, tap, free-fall and orientation detectors as
  configuration, the interrupt routing, the pins' form and the source
  registers.
- The self-test as a function that answers the change in milli-g, and the
  data sheet's window of 70 to 1500 mg.
- The suite runs over a double of the part on both buses and covers every line
  of `src/`.

### Known

- Hardware verification: NOT VERIFIED.  No board carries a LIS2DW12.
- The data sheet read is DS11811 Rev 8 (July 2019), the copy that could be
  fetched.  The brief named DS11445 and application note AN5038; DS11445 was
  not found as this part's number, and AN5038 could not be fetched (st.com
  refused the request).  Its mg weights for the tap and free-fall thresholds
  are therefore not used.
- The software trigger of the conversion on demand follows Rev 8's text
  (SLP_MODE_SEL and SLP_MODE_1 both written), which differs from ST's
  `lis2dw12-pid` driver (SLP_MODE_1 alone).  NOT VERIFIED on a part.
- The turn-on time after power-up is not given in Rev 8; the device program
  waits 20 ms.
- `AccelerometerAsync` and `ThermometerAsync` are not implemented: a `@value`
  struct takes no `async fn` (SPEC section 14.5).  `AsyncDeviceReady` is not
  implemented either, since no board's GPIO implements `AsyncEdge`; `irq_arm`
  is the part that does not depend on a board.
- The device program builds for the nRF52840 DK, the nRF54LM20 DK and the
  FRDM-MCXN947, with images of 15,544, 16,544 and 17,656 bytes of flash.  A
  reply from a LIS2DW12 over the Nordic buses is NOT VERIFIED until the driver
  rig.
- The SPI three-wire mode, the user offset registers, the high-pass filter
  path and the trigger of a conversion on demand from INT2 are not
  implemented.
