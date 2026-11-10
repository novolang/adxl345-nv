# adxl345-nv

A driver for Analog Devices' ADXL345, a three-axis accelerometer with an
I2C and an SPI interface, specified in Analog Devices' "ADXL345 Data
Sheet", document D07925-0-6/15(E), Rev. E, June 2015.  The driver
implements the `Accelerometer` and `Identifiable` traits of
[devices-nv](https://novo-lang.org/packages/devices-nv) and reads the
part in milli-g.

## Install

```sh
novo pkg add adxl345-nv
```

## Example

A device program reads the part over I2C.  The breakout's SDA and SCL go
to the board's I2C pins, its CS pin to the supply, which selects I2C, and
its SDO/ALT ADDRESS pin to ground, which puts the part at 0x53.  The
program waits for the part to start, brings it up at ±2 g in full
resolution at 100 Hz, and prints one line per sample over RTT.  With the
board flat and face up, `z` is close to 1000.

```novo norun:needs-pkg
use embedded_hal.{ BoardI2c, BoardGpio }
use dev_bus
use dev_accel.{ AccelMilliG, Accelerometer }
use adxl345
use adxl345.{ Adxl345 }

// The part's address with the ALT ADDRESS pin low.
const ADDR = 0x53

// A label and a number, written without building a string.
fn field(label: Str, v: Int) [hw]
    hal.uart.write(label)
    hal.uart.write(str.from_int(v))

@tier(embedded)
fn main() [hw]
    let part = Adxl345.attach(BoardI2c { id: 0 }, BoardGpio { id: 0 }, dev_bus.wiring_i2c(0, ADDR))
    hal.timer.delay_ms(2)
    if not part.begin()
        hal.uart.write("adxl345: no answer\n")
        return
    hal.timer.delay_ms(12)
    loop
        match part.accel_read_new()
            Some(a) =>
                hal.uart.write("adxl345")
                field(" x=", a.x)
                field(" y=", a.y)
                field(" z=", a.z)
                hal.uart.write("\n")
            None =>
                hal.timer.delay_ms(2)
```

The program is `tests/device_read.nv`.  Build it for a board with:

```sh
novo build --target=nrf52840-dk tests/device_read.nv
novo build --target=nrf54lm20-dk tests/device_read.nv
novo build --target=frdm-mcxn947 tests/device_read.nv
```

| Board | Flash | Of which the program | RAM at start |
|---|---|---|---|
| nRF52840 DK | 19,720 bytes | 4,674 bytes | 3,252 bytes |
| nRF54LM20 DK | 20,680 bytes | 4,840 bytes | 3,236 bytes |
| FRDM-MCXN947 | 22,008 bytes | 6,680 bytes | 2,748 bytes |

The Nordic DKs drive I2C with a TWIM (the nRF52840 DK's TWIM0 on SDA P0.26
and SCL P0.27, the nRF54LM20 DK's TWIM22 on SDA P1.03 and SCL P1.04) and SPI
with a SPIM, on the pins `docs/hal/README.md` names under "The Nordic DKs'
buses", so the three rows are the image with the driver's code in it.
Before the Nordic buses landed the two DK rows were 13,080 and 13,928
bytes, the image without the driver's reading path, because the board
packages answered for I2C and SPI as absent hardware.  A reply from the part
over a DK's bus is NOT VERIFIED until the driver rig carries one.  The rest of each image is the runtime, the compiler's support
library and tables.  The driver's own code allocates nothing: every buffer
is a fixed-size `Vec` in the caller's frame.

**Hardware: NOT VERIFIED.**  No board carries an ADXL345 yet.  The program
builds, and the driver's functions run over a double of the part that
follows the data sheet's register map and bus protocols.  A reading from a
part on a board has not been taken.

### The effects

Every function of the driver that reaches the bus declares `[hw]`, and
nothing else.  The driver performs no input or output of its own and
allocates nothing of its own.  The bus functions of devices-nv build one
short frame per transaction inside a `system` block and release it when
the call returns.  The module checks at every tier, `embedded` included.
The program above declares `[hw]` for the same reason.

## What it is

An accelerometer measures acceleration along three axes.  At rest it reads
the acceleration of gravity, 1 g, on the axis that points up, so it also
tells which way the board is tilted.

The ADXL345 measures within a range of ±2, ±4, ±8 or ±16 g.  Each axis is
a 16-bit two's complement word, and how many of its bits are valid depends
on two bits of DATA_FORMAT (Table 1).

| Mode | Valid bits at ±2 g | ±4 g | ±8 g | ±16 g | Step |
|---|---|---|---|---|---|
| Fixed 10-bit (FULL_RES clear) | 10 | 10 | 10 | 10 | 3.9, 7.8, 15.6 and 31.2 mg |
| Full resolution (FULL_RES set) | 10 | 11 | 12 | 13 | 3.9 mg at every range |

The step is the range's full scale over 2^(bits - 1), which is 256, 128,
64 and 32 counts per g in the fixed mode and 256 counts per g in full
resolution.  The data sheet prints 3.9 mg, the figure rounded from 1000 /
256; the driver converts with 256, so a count of 256 is 1000 mg.

The Justify bit puts the count at the top of the word instead of the
bottom.  The driver reads FULL_RES, Justify and the range in the same
transaction as the sample and shifts the count down, so a sample is
converted with the format it was taken in.

The part samples at a rate set by four bits of BW_RATE (Table 7).  Code
1010 is 100 Hz, each code below halves the rate down to 0000 at 0.098 Hz
(printed as 0.10), and 1111 is 3200 Hz.  LOW_POWER lowers the current at
12.5 to 400 Hz (Table 8), and the driver refuses it elsewhere.

The part has no status register.  A new sample sets DATA_READY in
INT_SOURCE, and the part can drive the same event onto its INT1 pin
(INT_ENABLE and INT_MAP), which the driver turns on.  The part also has a
FIFO of 32 samples, a self-test, and four detectors that run on the chip:
single tap and double tap, activity, inactivity and free fall.

The part has two bus interfaces on the same pins.  CS high selects I2C.

| | I2C | SPI |
|---|---|---|
| Address | 0x53, or 0x1D with ALT ADDRESS high | chip select low |
| Several registers in one read | the register address advances by itself | bit 6 of the command byte set (MB) |
| Read | register, repeated start, data | command byte with bit 7 set (R/W), then data |
| Clock | 400 kHz | 4 MHz, mode 3 (up to 5 MHz) |

The data sheet recommends an SPI clock of 2 MHz or more for 3200 Hz and
1600 Hz, and a bus clock in kHz of at least half the rate in Hz for the
others (p. 15).  At 400 kHz on I2C the highest usable rate is 800 Hz.

## What the package contains

| Module | What is in it |
|---|---|
| `adxl345` | The types `Adxl345` (I2C) and `Adxl345Spi` (SPI), the trait `Adxl345Port`, the bus framing, the register access, the conversions, the FIFO, the detectors, the interrupt routing, the self-test and the bring-up |

## How to choose an entry point

A program on a board uses `Adxl345` for a part on I2C and `Adxl345Spi` for
a part on SPI.  Each is built by `attach` with the board's bus, its GPIO
and the wiring, as a board's device table builds it.  Both implement
`Accelerometer` and `Identifiable`.

| Method | What it does |
|---|---|
| `attach(bus, pins, w)` | the part at `w.addr` (I2C) or with the chip select `w.cs` (SPI); nothing is sent |
| `begin()` | brings the part up, see below |
| `accel_read()`, `accel_ready()`, `accel_read_new()` | `Accelerometer`: the latest sample in milli-g, the data-ready flag, a sample not read before |
| `accel_range()`, `accel_set_range(r)` | the range, changed without touching the other bits of DATA_FORMAT |
| `accel_rate_hz()`, `accel_set_rate_hz(hz)` | the rate in whole hertz, and the slowest rate at or above `hz` |
| `accel_fifo_count()`, `accel_fifo_read(dst)` | the FIFO, which is off until `fifo_begin` |
| `device_id()` | `Identifiable`: DEVID, 0xE5 |
| `set_full_res(on)`, `set_justify(left)` | the resolution and the placement of the count |
| `rate_mhz()`, `set_rate_mhz(mhz)` | the rate in thousandths of a hertz, which has the sub-hertz rates |
| `low_power()`, `set_low_power(on)` | low-power mode, 12.5 to 400 Hz |
| `fifo_begin(n)`, `fifo_end()` | turn the FIFO on in stream mode with the watermark at `n` samples, and off |
| `configure_tap(...)`, `configure_activity(...)`, `configure_inactivity(...)`, `configure_free_fall(...)` | write the detectors' thresholds and times |
| `configure_interrupts(enable, map, active_low)` | route the flags to INT1 and INT2 |
| `interrupt_flags()`, `tap_status()` | INT_SOURCE and ACT_TAP_STATUS, both clear on read |
| `self_test()` | the change in milli-g that the part's self-test makes |
| `reg(r)`, `set_reg(r, v)` | one register, and a write that is read back |

`begin` checks DEVID, puts the part in standby, puts the registers in a
known state because the part has no reset pin and keeps them while powered
(FIFO bypassed, the detectors and the interrupts off, the offsets kept),
then sets full resolution at ±2 g, 100 Hz in normal power, the data-ready
on INT1 and measurement on.  The data sheet advises configuring in standby
and then enabling measurement (p. 12), which is the order.  It answers
`false` for a part that does not answer 0xE5 and for a register that does
not hold what was written.

A program that holds its own bus, or a test with a double of the part,
implements `Adxl345Port` over its bus and calls the module's functions,
which take the port: `adxl345.sample_read(port)`, `adxl345.start(port)`.
The framing of each bus is `i2c_read_into`, `i2c_write`, `spi_read_into`
and `spi_write`, generic over `I2cBus` and `SpiBus`.

### The detectors

The detection runs on the chip.  The driver writes the thresholds and times
in physical units and the part sets flags and pulses its pins.

| Function | Units | Registers |
|---|---|---|
| `tap_config(thresh_mg, dur_us, latent_us, window_us, axes)` | 62.5 mg, 625 us, 1.25 ms and 1.25 ms per count | THRESH_TAP, DUR, LATENT, WINDOW, TAP_AXES |
| `activity_config(thresh_mg, axes)` | 62.5 mg per count | THRESH_ACT, upper nibble of ACT_INACT_CTL |
| `inactivity_config(thresh_mg, time_s, axes)` | 62.5 mg and 1 s per count | THRESH_INACT, TIME_INACT, lower nibble of ACT_INACT_CTL |
| `free_fall_config(thresh_mg, time_ms)` | 62.5 mg and 5 ms per count | THRESH_FF, TIME_FF |

A threshold is held to 1 to 255 counts, because the data sheet warns that 0
may misbehave once the interrupt is on.  A zero DUR disables the single
tap, and a zero LATENT or WINDOW disables the double tap.  The axes are the
constants `TAP_X`, `TAP_Y`, `TAP_Z` and `TAP_SUPPRESS`, and `AXES_X`,
`AXES_Y`, `AXES_Z` and `AXES_AC` for activity and inactivity.  The data
sheet advises 300 to 600 mg and 100 to 350 ms for free fall.

`configure_interrupts(enable, map, active_low)` switches the interrupts off,
writes the map and the polarity, reads INT_SOURCE to clear what was
latched, and switches the chosen ones on, the order the data sheet advises
(p. 20).  `enable` and `map` are built from `INT_DATA_READY`,
`INT_SINGLE_TAP`, `INT_DOUBLE_TAP`, `INT_ACTIVITY`, `INT_INACTIVITY`,
`INT_FREE_FALL`, `INT_WATERMARK` and `INT_OVERRUN`; a bit set in `map`
routes that flag to INT2.

## The rules a user needs

1. Wait for the part to turn on before `begin`.  VDD I/O must be present
   first, or the part holds the bus in conflict (Table 6).  After
   `begin`, the first sample comes about 1.1 ms plus one sample period
   later, 11 ms at 100 Hz (Table 1, note 7).
2. `begin` has to run before a reading: the part is in standby at
   power-on.
3. A reading is converted with the format it was taken in.  Samples
   already in the FIFO when the range changes keep the old format, and are
   converted with the new one; empty the FIFO with `fifo_end` and
   `fifo_begin` around a change.
4. `accel_ready` reads INT_SOURCE, and a read of INT_SOURCE clears the
   latched tap, activity, inactivity and free-fall flags (p. 20).  A
   program that uses the detectors reads the flags with `interrupt_flags`
   and takes DATA_READY from the same byte, or waits on the INT1 pin.
5. INT_SOURCE reads 0x02 at reset: the watermark flag is set while FIFO_CTL
   holds 0 samples.  The flag matters only once WATERMARK is enabled.
6. LOW_POWER is refused outside 12.5 to 400 Hz, and `accel_set_rate_hz`
   clears it when the new rate is outside that range.  The rate taken is
   rarely the rate asked for, and the sub-hertz rates read as 1 Hz or more
   in whole hertz; `rate_mhz` has them exactly.
7. The FIFO holds 32 samples and the part reports up to 33, because the
   output registers hold one more.  `accel_fifo_count` answers at most 32.
   A sample is read as one burst of the six data registers, because any
   read of the FIFO takes a level out of it, and a single-byte read would
   lose the rest of the sample (p. 27).  The data sheet asks for 5 us
   between the end of one FIFO read and the start of the next, or of a
   FIFO_STATUS read.  Two transactions are further apart than that on both
   buses, since chip select rises between them.
8. `fifo_begin` puts the watermark on INT1 in place of the data-ready, and
   `fifo_end` puts the data-ready back and empties the FIFO.
9. The self-test needs the board still, normal power, 100 to 800 Hz or
   3200 Hz, and a range of at least 8 g.  The driver sets 100 Hz, full
   resolution at ±16 g, averages ten samples after four have settled,
   switches the force on, and does the same again (p. 31).  The change it
   answers is in milli-g.  Table 1 gives 0.20 to 2.10 g on X, -2.10 to
   -0.20 g on Y and 0.30 to 3.40 g on Z at a supply of 2.5 V, and Table 14
   scales them for other supplies.  `self_test_within(d, xy_permille,
   z_permille)` compares a change with them, with the factors in
   thousandths: 1000 and 1000 at 2.5 V, 640 and 800 at 2.0 V.
10. A double tap needs a window at least as long as the latency to see a
    second tap.  The sheet states that the second tap must start after the
    latency and within the window.
11. `device_id` answers `None` for a bus that no part drives: an SPI bus
    reads 0xFF, or 0x00 when held low.  The ADXL346 answers 0xE6 and is not
    accepted.

## What is not included

- The SPI three-wire mode: the SPI bit of DATA_FORMAT stays 0.
- The offset registers OFSX, OFSY and OFSZ, which hold a calibration, and
  the sleep, auto-sleep, link and wakeup bits of POWER_CTL.  `reg` and
  `set_reg` reach them.
- The FIFO's trigger and plain FIFO modes: the FIFO runs in stream mode.
- The ADXL346, whose DEVID is 0xE6 and which has orientation sensing.
- The async form of the traits, which an `async fn` cannot take as a value
  type.

## Related packages

- [devices-nv](https://novo-lang.org/packages/devices-nv) names the
  traits this driver implements, the wiring record and the register
  helpers it uses.
- [embedded-hal-nv](https://novo-lang.org/packages/embedded-hal-nv) names
  the `I2cBus`, `SpiBus` and `GpioOut` traits and the board handles.
- lis3dh-nv drives ST's LIS3DH, an accelerometer with the same bus
  conventions in other places: the SPI increment bit is bit 6 there too.
- lsm6dso-nv drives a six-axis part that implements `Accelerometer` and
  `Gyroscope`.

## Tests

`novo test` runs the module over a double of the part that answers on an
I2C bus and on an SPI bus: the command byte and the MB bit, the advancing
register address, DEVID, every conversion at every range with and without
full resolution and the Justify bit, the rate codes, low-power mode, the
range and format changes, the FIFO, the thresholds and times, the
detectors and the interrupt routing, the self-test, the bring-up and its
refusals, and the two types over the host's board handles.  The log of
bytes sent is compared for each entry point.  A second port refuses a
chosen write of a chosen register, which reaches the refusals a read-only
register cannot.  The suite covers every line of `src/`.

**Hardware: NOT VERIFIED.**  See the example section.

The sources read for this driver, for their fixes and not ported, are the
data sheet above, Zephyr's `drivers/sensor/adi/adxl345` and the Linux
kernel's `drivers/iio/accel/adxl345_core.c`.  Embassy and esp-hal hold no
driver for the part.  Each fix kept is named in `src/adxl345.nv` where it
is kept.

## Licence

Apache-2.0.
