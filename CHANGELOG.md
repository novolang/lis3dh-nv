# Changelog

## 0.2.0 — 2026-11-24

The register reads and writes, and the bus set-up, are devices-nv's `RegPort` and `wiring_*` functions, and `Lis3dhPort` is smaller by them.  The release depends on devices-nv 0.4.1.

### Breaking

- **`Lis3dhPort` has the supertrait `RegPort` and no longer declares `read_regs` and `write_reg`.**  A type that implements `Lis3dhPort` for a bus of its own moves those two methods, unchanged, into an `impl RegPort`.  *Migration:* with `use dev_bus.{ RegPort }`,

  ```novo
  impl RegPort for MyPort
      fn read_regs(self, reg: Int, n: Int, var dst: DevBytes) -> Bool [hw]
          ...
      fn write_reg(self, reg: Int, v: Int) -> Bool [hw]
          ...

  impl Lis3dhPort for MyPort
      ...                      // the other methods stay where they are
  ```

  The package's own types are migrated.  A caller that holds one of them is unaffected.

### Deprecated

- `i2c_setup`, `spi_setup`, `irq_level_of`, `reg_read`, `reg_write`, `reg_update`: each is a one-line function over the method or the `wiring_*` function that replaces it, warns W1001 at a call with the replacement named, and is removed in the next release.  `Lis3dhPort` itself is the driver's trait and stays.

### Changed

- The register helpers the module carried (`reg_read`, `reg_write`, `reg_update` over its port) are devices-nv's `RegPort` methods (`read_reg`, `write_reg_verified`, `update_reg_verified` and the others); the module calls them as `p.read_reg(reg)`.
- The bus set-up and the interrupt-level read are devices-nv's `wiring_i2c_setup`, `wiring_spi_setup` and `wiring_irq_level`, called with the part's rate and mode.
- Image size on nrf52840-dk, `.text` plus read-only data, before and after: 15,428 bytes before and 15,364 after on the read-out (nrf52-qemu: 8,400 and 8,400).

## 0.1.1 — 2026-11-23

The package asks for embedded-hal-nv ^0.4.0 (was ^0.3.2) and devices-nv ^0.4.0 (was ^0.3.2), the minor of both trait packages that the other drivers and the board packages ask for, so one version of each serves a program that takes this package beside any of them.  Under the older ranges, a program that named this package and a driver on 0.4 stopped with "no version of 'embedded-hal-nv' satisfies every requirement".

No signature of the package changed; the suite passes without an edit.

The device program builds for the boards its README names, flash nrf52840-dk 15,376 B before and 15,376 B after; nrf54lm20-dk 16,468 B before and 16,468 B after.

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
- The device program builds for the nRF52840 DK and the nRF54LM20 DK.  Their
  board packages answered for I2C and SPI as absent hardware when the
  package was written (images of 13,088 and 13,932 bytes of flash, without
  the reading path); the TWIM and SPIM drivers, tracked by
  `features/embedded-devex/nordic-twim-i2c-driver.md`, landed on 2026-11-10,
  and the images are now 19,728 and 20,684 bytes with the reading path.  A
  reply from a LIS3DH over those buses is NOT VERIFIED until the driver rig.
