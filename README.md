# lis2dw12-nv

A driver for ST's LIS2DW12, a three-axis accelerometer with an I2C and an
SPI interface, specified in ST's data sheet "LIS2DW12: MEMS digital output
motion sensor: high-performance ultra-low-power 3-axis femto accelerometer",
DS11811 Rev 8, July 2019.  The driver implements the `Accelerometer`,
`Thermometer` and `Identifiable` traits of
[devices-nv](https://novo-lang.org/packages/devices-nv) and reads the part in
milli-g.

## Install

```sh
novo pkg add lis2dw12-nv
```

## Example

A device program reads the part over I2C.  The breakout's SDA and SCL go to
the board's I2C pins, its SDO/SA0 pin to ground, which puts the part at
0x18, and its CS pin to the supply, which selects I2C.  The program waits 20
ms for the part to turn on (the data sheet states no figure), brings it up
at ±2 g in High-Performance Mode at 100 Hz, and prints one line per sample
over RTT.  With the board flat and face up, `z` is close to 1000.

```novo norun:needs-pkg
use embedded_hal.{ BoardI2c, BoardGpio }
use dev_bus
use dev_accel.{ AccelMilliG, Accelerometer }
use dev_thermo.{ TempMilliC, Thermometer }
use lis2dw12
use lis2dw12.{ Lis2dw12 }

// The part's address with SA0 low.
const ADDR = 0x18

// A label and a number, written without building a string.
fn field(label: Str, v: Int) [hw]
    hal.uart.write(label)
    hal.uart.write(str.from_int(v))

@tier(embedded)
fn main() [hw]
    let part = Lis2dw12.attach(BoardI2c { id: 0 }, BoardGpio { id: 0 }, dev_bus.wiring_i2c(0, ADDR))
    hal.timer.delay_ms(20)
    if not part.begin()
        hal.uart.write("lis2dw12: no answer\n")
        return
    loop
        match part.accel_read_new()
            Some(a) =>
                hal.uart.write("lis2dw12")
                field(" x=", a.x)
                field(" y=", a.y)
                field(" z=", a.z)
                match part.temp_read()
                    Some(t) => field(" t=", t.milli_c)
                    None    => hal.uart.write(" t=?")
                hal.uart.write("\n")
            None    =>
                hal.timer.delay_ms(2)
```

The program is `tests/device_read.nv`, and `tests/device_build.sh` builds it
for the boards below.  The nRF52840 DK's I2C bus is TWIM0 on SDA P0.26 and
SCL P0.27, and the nRF54LM20 DK's is TWIM22 on SDA P1.03 and SCL P1.04
(`docs/hal/README.md` in the novo-lang repository, "The Nordic DKs' buses").

```sh
novo build --target=nrf52840-dk tests/device_read.nv
novo build --target=nrf54lm20-dk tests/device_read.nv
novo build --target=frdm-mcxn947 tests/device_read.nv
```

**Hardware: NOT VERIFIED.**  No board carries a LIS2DW12 yet.  The program
builds for the three boards, the image for the emulated board runs under
QEMU and reports that the part does not answer, and the driver's functions
run over a double of the part that follows the data sheet's register map and
bus protocols.  A reading from a part on a board has not been taken, and a
reply from a LIS2DW12 over the Nordic buses is NOT VERIFIED until the driver
rig exists.

### The tier, the boards and the footprint

The package is checked at the embedded tier (`[tier]` in `novo.toml`), so
every function in it is also valid at `rt` and `system`; no migration knob is
on.  Each row is the memory report of `novo build` for the program above, with
the driver's reading path in the image.

| Board | Flash | Of which the program | `.data` and `.bss` | Static | Arena provable |
|---|---|---|---|---|---|
| nRF52840 DK | 15,544 bytes | 4,948 bytes | 3,252 bytes | 510 bytes | 496 bytes |
| nRF54LM20 DK | 16,544 bytes | 5,120 bytes | 3,236 bytes | 510 bytes | 496 bytes |
| FRDM-MCXN947 | 17,656 bytes | 6,796 bytes | 2,748 bytes | not recorded | not recorded |

The rest of each image is the runtime, the compiler's support library and
tables.  The driver's own code allocates nothing: every buffer is a
fixed-size `Vec` in the caller's frame, and `AccelMilliG`, `TempMilliC` and
`TapConfig` are `@value` structs.  "Arena provable" is the compiler's count of
allocations at initialisation.

### The effects

Every function of the driver that reaches the bus declares `[hw]`, and
nothing else.  The driver performs no input or output of its own and
allocates nothing of its own.  The bus functions of devices-nv build one
short frame per transaction inside a `system` block and release it when the
call returns.  The two awaiting functions, `accel_measure` and `accel_next`,
declare `[hw, async]` and are `@tier(rt)`.  The program above declares `[hw]`
for the same reason.

## What it is

An accelerometer measures acceleration along three axes.  At rest it reads
the acceleration of gravity, 1 g, on the axis that points up, so it also
tells which way the board is tilted.

The LIS2DW12 measures within a full scale of ±2, ±4, ±8 or ±16 g.  Each
axis is a 16-bit two's complement word, left-justified.  How many of its
bits are valid depends on the mode (data sheet sections 3.2.1 and 8.12).

| Mode | Valid bits | Step at ±2 g | ±4 g | ±8 g | ±16 g |
|---|---|---|---|---|---|
| High-Performance Mode | 14 | 0.244 mg | 0.488 mg | 0.976 mg | 1.952 mg |
| Low-Power Modes 2, 3 and 4 | 14 | 0.244 mg | 0.488 mg | 0.976 mg | 1.952 mg |
| Low-Power Mode 1 | 12 | 0.976 mg | 1.952 mg | 3.904 mg | 7.808 mg |

The figures are Table 3's sensitivity.  A reading is converted with the mode,
the low-power level and the range that the part has when the sample is read:
the driver reads CTRL1 to OUT_Z_H in one transaction.  Because the words are
left-justified, the same acceleration is the same word in 12 and in 14 bits,
and the width decides only how many of the low bits carry information.
`bits_of_ctrl1` and `Lis2dw12.resolution()` say which width a setting gives.
The part has four low-power levels that trade noise against current, a
low-noise setting, and a digital filter whose bandwidth is ODR/2, ODR/4,
ODR/10 or ODR/20.

The part samples at a rate set by four bits of CTRL1 (Table 29), and the
codes mean other rates in the two families of modes.

| ODR code | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 |
|---|---|---|---|---|---|---|---|---|---|
| High-Performance Mode | 12.5 Hz | 12.5 Hz | 25 Hz | 50 Hz | 100 Hz | 200 Hz | 400 Hz | 800 Hz | 1600 Hz |
| Low-power modes | 1.6 Hz | 12.5 Hz | 25 Hz | 50 Hz | 100 Hz | 200 Hz | 200 Hz | 200 Hz | 200 Hz |

Code 0 is power-down.  A new sample sets DRDY in STATUS.  The part can drive
the same event onto its INT1 pin (CTRL4_INT1_PAD_CTRL, INT1_DRDY), and the
driver turns that on.  The die temperature is in OUT_T_L and OUT_T_H as a
12-bit word, 16 counts to the degree, 0 at 25 °C with an offset of ±15 °C
(Table 5), so it shows change and not an absolute temperature.

The part has two bus interfaces on the same pins.  CS high selects I2C.

| | I2C | SPI |
|---|---|---|
| Address | 0x18, or 0x19 with SDO/SA0 high | chip select low |
| Several registers in one read | automatic while CTRL2's IF_ADD_INC is set | automatic while CTRL2's IF_ADD_INC is set |
| Read | sub-address, repeated start, data | command byte with bit 7 set (RW), then data |
| Clock | 400 kHz | 4 MHz, mode 3 (up to 10 MHz) |

Unlike the other ST sensors of this family, neither bus has an increment bit
of its own: IF_ADD_INC, which is set at reset, decides.  `begin` sets it, and a
program that clears it makes a burst read return one register repeatedly.

## What the package contains

| Module | What is in it |
|---|---|
| `lis2dw12` | The types `Lis2dw12` (I2C) and `Lis2dw12Spi` (SPI), `Lis2dwMode`, `TapConfig`, the trait `Lis2dw12Port`, the bus framing, the register access, the conversions, the FIFO, the conversion on demand, the detectors, the self-test, the awaiting functions and the bring-up |

## How to choose an entry point

A program on a board uses `Lis2dw12` for a part on I2C and `Lis2dw12Spi` for
a part on SPI.  Each is built by `attach` with the board's bus, its GPIO and
the wiring, as a board's device table builds it.  Both implement
`Accelerometer`, `Thermometer` and `Identifiable`.

| Method | What it does |
|---|---|
| `attach(bus, pins, w)` | the part at `w.addr` (I2C) or with the chip select `w.cs` (SPI); nothing is sent |
| `begin()` | brings the part up, see below |
| `accel_read()`, `accel_ready()`, `accel_read_new()` | `Accelerometer`: the latest sample in milli-g, the data-ready flag, a sample not read before |
| `accel_range()`, `accel_set_range(r)` | the full scale, changed without touching the other bits of CTRL6 |
| `accel_rate_hz()`, `accel_set_rate_hz(hz)` | the rate in whole hertz (1.6 Hz reads 1, 12.5 Hz reads 12), and the slowest rate at or above `hz` in the present mode |
| `accel_fifo_count()`, `accel_fifo_read(dst)` | the FIFO, which is off until `fifo_begin` |
| `temp_start()`, `temp_read()` | `Thermometer`: the part converts at its sample rate, so a start answers 0, or -1 when the part is in power-down |
| `device_id()` | `Identifiable`: WHO_AM_I, 0x44 |
| `mode()`, `lp_level()`, `resolution()`, `set_mode(m, level)` | the mode, the low-power level 1 to 4, the width in bits, and the change of mode and level that leaves the rate |
| `fifo_begin(n)`, `fifo_end()` | turn the FIFO on in continuous mode with the threshold at `n` samples, and off |
| `convert()` | one conversion on demand: triggers it, polls until it ends, answers the sample |
| `self_test(negative)` | the change in milli-g that the part's self-test makes |
| `reg(r)`, `set_reg(r, v)` | one register, and a write that is read back |

The detectors, the interrupt routing and the rest are functions of the
module that take the port, so they work for both types:
`lis2dw12.wake_up_config(part, 500, 2)`.

| Function | What it sets |
|---|---|
| `wake_up_config(p, thr_mg, dur)` | the wake-up threshold in milli-g at the present range, and the duration in sample periods |
| `activity_config(p, sleep_on, stationary, sleep_dur)` | the change to 12.5 Hz when the part is still, or the detection that leaves the rate alone |
| `tap_config(p, c)` | single and double tap from a `TapConfig`: the three thresholds, the priority, the axes, the latency, the quiet time and the shock time |
| `free_fall_config(p, code, dur)` | the free-fall threshold code 0 to 7 and the duration 0 to 63 |
| `orientation_config(p, four_d, code)` | 6D or 4D detection and the angle |
| `int1_route(p, bits)`, `int2_route(p, bits)` | which events reach INT1 and INT2 (Tables 36 and 38) |
| `interrupts_enable(p, on)`, `int_pins(p, ...)`, `drdy_pulsed(p, on)` | the detectors on or off, the pins' polarity, drive and latching, and the data-ready as a pulse |
| `tap_source`, `wake_up_source`, `orientation_source`, `event_source` | the source registers, read whole |
| `soft_reset(p)`, `reboot(p)`, `low_noise_write(p, on)`, `bandwidth_write(p, code)` | the soft reset, the reload of the trimming from non-volatile memory, the low-noise setting and the filter bandwidth |

A program that holds its own bus, or a test with a double of the part,
implements `Lis2dw12Port` over its bus and calls the module's functions, which
take the port: `lis2dw12.sample_read(port)`, `lis2dw12.start(port)`.  The
framing of each bus is `i2c_read_into`, `i2c_write`, `spi_read_into` and
`spi_write`, generic over `I2cBus` and `SpiBus`.

### Waiting for a sample

`AccelerometerAsync` cannot be implemented by `Lis2dw12`: a `@value` struct
takes no `async fn` (SPEC section 14.5).  The package gives the same waits as
functions for a host's scheduler, each taking a `TimerAsync`:

- `accel_measure(part, timer)` triggers one conversion on demand, parks the
  task for the turn-on time of the low-power level (1.20, 1.70, 2.30 or 3.55
  ms), and answers the sample.  The part measures in the background and goes
  back to power-down, so the task is free in between.
- `accel_next(part, timer, period_us, tries)` parks the task between polls of
  the data-ready flag.

A program at `@tier(embedded)` waits on the INT1 line instead.  `irq_arm(edge,
w, f)` and `irq_disarm(edge, w)` arm and release a rising-edge wait on the pin
of the wiring through any `AsyncEdge` source.  No board's GPIO implements
`AsyncEdge` yet, so this path is NOT VERIFIED on a board.

## The rules a user needs

1. Call `begin` before a reading: the part is in power-down at power-on.  It
   resets the part, because the part keeps its registers while powered, and
   writes CTRL1 last.
2. A reading is converted at the range, the mode and the low-power level it was
   taken in.  After a change of rate or mode the part settles; in
   High-Performance Mode at ODR/2 bandwidth the first sample at 100 Hz and up
   is to be discarded (Table 13), and so is the first sample after a switch
   into or out of the FIFO (section 5.3).
3. `accel_set_rate_hz` answers the rate taken, which is rarely the rate asked
   for, and in whole hertz.  `rate_write` answers millihertz.  The low-power
   modes stop at 200 Hz; a request above it takes code 6.
4. With the FIFO on, a read of OUT_X_L takes the oldest sample out of the
   FIFO.  `accel_read` therefore takes a sample from the FIFO while it is on,
   and the FIFO's count falls by one.
5. A conversion on demand needs `single_arm` first: the part goes to the
   single-conversion mode with the software trigger selected.  See the open
   question below.
6. The self-test change is the second reading minus the first, with the part
   at ±4 g in High-Performance Mode at 50 Hz.  Table 3 gives the positive
   difference as 70 to 1500 mg, and `self_test_ok` checks a change against
   that window.
7. The tap and free-fall thresholds are the data sheet's own fields and
   scale with the full scale.  The mg weight of each step is in application
   note AN5038, which could not be fetched, so the driver gives none; the
   wake-up threshold has a weight in the data sheet, 1/64 of the full scale,
   and is given in milli-g.
8. `device_id` answers `None` for a bus that no part drives: an SPI bus reads
   0xFF, or 0x00 when held low.
9. `temp_read` is a change from the die's own offset of ±15 °C, not an
   absolute temperature.

### An open question

The data sheet (Rev 8, section 3.2.2 and CTRL3, Table 34) says the software
trigger is SLP_MODE_SEL set and SLP_MODE_1 written to 1.  ST's own driver
(`lis2dw12-pid`, `lis2dw12_data_rate_set`) writes SLP_MODE_1 alone for the
software trigger and SLP_MODE_SEL alone for the pin trigger, which is the
opposite.  The driver follows the data sheet and keeps the two bits in two
named constants, `CTRL3_SLP_SEL` and `CTRL3_SLP_1`.  Which is right is NOT
VERIFIED until the part is on a bench.

## What is not included

- The SPI three-wire mode (CTRL2's SIM): the 4-wire mode is implemented.
- The user offset registers (X_OFS_USR, Y_OFS_USR and Z_OFS_USR), the
  high-pass filter path (CTRL6's FDS) and the high-pass reference mode.
  `reg` and `set_reg` reach them.
- The trigger of a conversion on demand from a clock on INT2; the software
  trigger is implemented.
- The Android stationary detection beyond the `stationary` bit of
  `activity_config`, and the interplay of the sleep state with the ODR.
- The mg weight of the tap and free-fall thresholds (see the rules).
- The other parts of ST's family, which are separate data sheets with their
  own WHO_AM_I values and register maps: the LIS2DH12, LIS2DE12, LIS2DS12,
  LIS2DTW12 and IIS2DLPC.  This driver is for the LIS2DW12.
- A part with the address pin wired to neither the supply nor ground: SA0
  selects 0x18 or 0x19, and the wiring record carries the one the board uses.

## Related packages

- [devices-nv](https://novo-lang.org/packages/devices-nv) names the traits this
  driver implements, the wiring record and the register helpers it uses.
- [embedded-hal-nv](https://novo-lang.org/packages/embedded-hal-nv) names the
  `I2cBus`, `SpiBus`, `GpioOut`, `TimerAsync` and `AsyncEdge` traits and the
  board handles.
- [lis3dh-nv](https://novo-lang.org/packages/lis3dh-nv) drives the LIS3DH, an
  accelerometer of the same maker with another register map.
- [adxl345-nv](https://novo-lang.org/packages/adxl345-nv) drives the ADXL345 of
  Analog Devices.

## Tests

`novo test` runs the module over a double of the part that answers on an I2C
bus and on an SPI bus: the sub-address and the command byte, the increment, the
conversions at every mode, width and range, the rates, the range and mode
changes, the temperature, the FIFO, the conversion on demand and the two
awaiting functions over a timer double, the self-test, the detectors, the
bring-up and its refusals, and the two types over the host's board handles.
The log of bytes sent is compared for each entry point.  The suite covers every
line of `src/`.

**Hardware: NOT VERIFIED.**  See the example section.

The sources read for this driver, for their fixes and not ported, are the ST
data sheet above (the copy fetched is DS11811 Rev 8; the number DS11445 and
application note AN5038 named in the brief could not be matched to the part or
fetched), Zephyr's `drivers/sensor/st/lis2dw12`, ST's `lis2dw12-pid` and the
`lis2dw12` crate 0.1.3.  Each fix kept is named in `src/lis2dw12.nv` where it
is kept:

- the crate writes its low-noise bit into CTRL1, where bit 2 is MODE0; the bit
  is CTRL6's;
- the crate sets and clears IF_ADD_INC around every multi-byte read; the
  driver sets it once, in `begin`;
- the crate converts every sample with one scale, and Zephyr's driver fixes
  its gain once at initialisation from the configured power mode, so a later
  change of mode leaves the gain wrong; this driver converts with the width
  and the range read in the same transaction as the sample;
- Zephyr waits a fixed 100 microseconds after the soft reset; the driver polls
  the self-clearing bit;
- Zephyr makes the pulsed or latched data-ready a setting of its own
  (`drdy-pulsed`), since a latched line that is not read stays high and hides
  the next sample; `drdy_pulsed` is the same setting here.

## Licence

Apache-2.0.
