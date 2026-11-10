# Changelog

## 0.1.0

The first release: a driver for Analog Devices' ADXL345 three-axis
accelerometer on an I2C or an SPI bus.

- `Adxl345` (I2C) and `Adxl345Spi` (SPI), the types a board's device table
  builds, implementing devices-nv's `Accelerometer`, including the FIFO,
  and `Identifiable`, with their own methods for full resolution, the
  Justify bit, the rate in thousandths of a hertz, low-power mode, the
  detectors, the interrupt routing, the self-test and register access.
- The register access as functions over `Adxl345Port`, a trait that a type
  holding a bus implements, and the framing of each bus as functions over
  `I2cBus` and `SpiBus`.
- Readings in milli-g at 256 counts per g in full resolution and 256, 128,
  64 and 32 counts per g in the fixed 10-bit mode, converted with the
  format read in the same transaction as the sample, including the
  left-justified count.
- The output data rates from 0.098 Hz to 3200 Hz, and low-power mode,
  refused outside 12.5 to 400 Hz.
- The FIFO as an option: off after `begin`, on in stream mode through
  `fifo_begin`, read as bursts of six registers, capped at 32 samples.
- The tap, activity, inactivity and free-fall detectors as configuration
  in milli-g and in microseconds, milliseconds or seconds, and the
  interrupt routing in the order the data sheet advises.
- The self-test as a function that answers the change in milli-g, and a
  comparison with Table 1's limits scaled by Table 14's factors.
- The suite runs over a double of the part on both buses and covers every
  line of `src/`.

### Known

- Hardware verification: NOT VERIFIED.  No board carries an ADXL345.
- The data sheet read is Rev. E (June 2015), the revision that could be
  fetched.  Later revisions exist and should be read against it.
- The device program builds for the nRF52840 DK and the nRF54LM20 DK, whose
  board packages answer for I2C and SPI as absent hardware: on those images
  `begin` answers `false` until the boards drive the buses.  The gap is
  tracked by `features/embedded-devex/nordic-twim-i2c-driver.md`.
- The SPI three-wire mode, the offset registers, the sleep modes and the
  FIFO's trigger mode are not implemented.
