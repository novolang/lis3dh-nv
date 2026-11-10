# Changelog

## 0.1.0

The first release: a driver for ST's LIS3DH three-axis accelerometer on an
I2C or an SPI bus.

- `Lis3dh` (I2C) and `Lis3dhSpi` (SPI), the types a board's device table
  builds, implementing devices-nv's `Accelerometer`, including the FIFO,
  and `Identifiable`, with their own methods for the resolution mode, the
  FIFO, the self-test and register access.
- The register access as functions over `Lis3dhPort`, a trait that a type
  holding a bus implements, and the framing of each bus as functions over
  `I2cBus` and `SpiBus`.
- Readings in milli-g at the sensitivity of the data sheet's Table 3 in
  the 12-bit mode, four times as far in the 10-bit mode and sixteen times
  in the 8-bit mode, converted with the settings read in the same
  transaction as the sample.
- The FIFO as an option: off after `begin`, on in stream mode through
  `fifo_begin`.
- The self-test as a function that answers the change in milli-g, and a
  comparison with limits the caller gives.
- The suite runs over a double of the part on both buses and covers every
  line of `src/`.

### Known

- Hardware verification: NOT VERIFIED.  No board carries a LIS3DH.
- The data sheet read is Doc ID 17530 Rev 1 (May 2010), the revision that
  could be fetched.  Its output data rates for code 1001 and 1000 are
  rounded (1.25 kHz, 5 kHz, 1.6 kHz), and the driver reports 1344, 5376
  and 1620 Hz from Zephyr's table.  Its self-test gives typical values and
  no limits.  A later revision should be read against both.
- The device program builds for the nRF52840 DK and the nRF54LM20 DK, whose
  board packages answer for I2C and SPI as absent hardware: on those
  images `begin` answers `false` until the boards drive the buses.  The gap is tracked by
  `features/embedded-devex/nordic-twim-i2c-driver.md`.
