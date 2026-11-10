# lis3dh-nv

A driver for ST's LIS3DH, a three-axis accelerometer with an I2C and an
SPI interface, specified in ST's data sheet "LIS3DH: MEMS digital output
motion sensor, ultra low-power high performance 3-axes nano
accelerometer", Doc ID 17530 Rev 1, May 2010.  The driver implements the
`Accelerometer` and `Identifiable` traits of
[devices-nv](https://novo-lang.org/packages/devices-nv) and reads the
part in milli-g.

## Install

```sh
novo pkg add lis3dh-nv
```

## Example

A device program reads the part over I2C.  The breakout's SDA and SCL go
to the board's I2C pins, its SDO/SA0 pin to ground, which puts the part
at 0x18.  The program waits the 5 ms the part takes to boot, brings it
up at ±2 g in high resolution at 100 Hz, and prints one line per sample
over RTT.  With the board flat and face up, `z` is close to 1000.

```novo norun:needs-pkg
use embedded_hal.{ BoardI2c, BoardGpio }
use dev_bus
use dev_accel.{ AccelMilliG, Accelerometer }
use lis3dh
use lis3dh.{ Lis3dh }

// The part's address with SA0 low.
const ADDR = 0x18

// A label and a number, written without building a string.
fn field(label: Str, v: Int) [hw]
    hal.uart.write(label)
    hal.uart.write(str.from_int(v))

@tier(embedded)
fn main() [hw]
    let part = Lis3dh.attach(BoardI2c { id: 0 }, BoardGpio { id: 0 }, dev_bus.wiring_i2c(0, ADDR))
    hal.timer.delay_ms(5)
    if not part.begin()
        hal.uart.write("lis3dh: no answer\n")
        return
    loop
        match part.accel_read_new()
            Some(a) =>
                hal.uart.write("lis3dh")
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
```

| Board | Flash | Of which the program | RAM at start |
|---|---|---|---|
| nRF52840 DK | 13,088 bytes | 1,368 bytes | 2,660 bytes |
| nRF54LM20 DK | 13,932 bytes | 1,422 bytes | 2,644 bytes |

The rest of the flash is the runtime, the compiler's support library and
tables.  The driver allocates nothing: every buffer is a fixed-size
`Vec` in the caller's frame.

**Hardware: NOT VERIFIED.**  No board carries a LIS3DH yet.  The program
builds, and the driver's functions run over a double of the part that
follows the data sheet's register map and bus protocols.  A reading from
a part on a board has not been taken.

### The effects

Every function of the driver that reaches the bus declares `[hw]`, and
nothing else.  The driver performs no input or output of its own and
allocates nothing, so the module checks at every tier, `embedded`
included.  The program above declares `[hw]` for the same reason.

## What it is

An accelerometer measures acceleration along three axes.  At rest it
reads the acceleration of gravity, 1 g, on the axis that points up, so
it also tells which way the board is tilted.

The LIS3DH measures within a full scale of ±2, ±4, ±8 or ±16 g.  Each
axis is a 16-bit two's complement count, left-justified: the valid bits
are the top ones, and how many depends on the mode (data sheet
section 3.2.1).

| Mode | Valid bits | Step at ±2 g | ±4 g | ±8 g | ±16 g |
|---|---|---|---|---|---|
| High resolution (HR set) | 12 | 1 mg | 2 mg | 4 mg | 12 mg |
| Normal (HR and LPen clear) | 10 | 4 mg | 8 mg | 16 mg | 48 mg |
| Low power (LPen set) | 8 | 16 mg | 32 mg | 64 mg | 192 mg |

The 12-bit figures are Table 3's sensitivity; the others are four and
sixteen times as large.  The ±16 g step is 12 mg, not 16 g over 2^11.
The driver reads the mode and the range in the same transaction as the
sample, so a reading is converted with the settings it was taken in.

The part samples at a rate set by four bits of CTRL_REG1 (Table 26).

| Rate | 1 Hz | 10 Hz | 25 Hz | 50 Hz | 100 Hz | 200 Hz | 400 Hz | 1620 Hz | 1344 Hz | 5376 Hz |
|---|---|---|---|---|---|---|---|---|---|---|
| ODR code | 0001 | 0010 | 0011 | 0100 | 0101 | 0110 | 0111 | 1000 | 1001 | 1001 |
| Mode | all | all | all | all | all | all | all | low power | normal, high resolution | low power |

A new sample sets ZYXDA in STATUS_REG.  The part can drive the same
event onto its INT1 pin (CTRL_REG3, I1_DRDY1), and the driver turns that
on.  The part also has a FIFO of 32 samples, a self-test, interrupt
generators for motion and free fall, and a temperature sensor.

The part has two bus interfaces on the same pins.  Chip select high
selects I2C.

| | I2C | SPI |
|---|---|---|
| Address | 0x18, or 0x19 with SDO/SA0 high | chip select low |
| Several registers in one read | bit 7 of the sub-address set | bit 6 of the command byte set (MS) |
| Read | sub-address, repeated start, data | command byte with bit 7 set (RW), then data |
| Clock | 400 kHz | 4 MHz, mode 3 (up to 10 MHz) |

The two buses put the increment bit in different places.  On SPI, bit 7
is RW, so a driver that copies the I2C sub-address onto SPI reads and
writes the wrong way round.

## What the package contains

| Module | What is in it |
|---|---|
| `lis3dh` | The types `Lis3dh` (I2C) and `Lis3dhSpi` (SPI), `Lis3dhMode`, the trait `Lis3dhPort`, the bus framing, the register access, the conversions, the FIFO, the self-test and the bring-up |

## How to choose an entry point

A program on a board uses `Lis3dh` for a part on I2C and `Lis3dhSpi` for
a part on SPI.  Each is built by `attach` with the board's bus, its GPIO
and the wiring, as a board's device table builds it.  Both implement
`Accelerometer` and `Identifiable`.

| Method | What it does |
|---|---|
| `attach(bus, pins, w)` | the part at `w.addr` (I2C) or with the chip select `w.cs` (SPI); nothing is sent |
| `begin()` | brings the part up, see below |
| `accel_read()`, `accel_ready()`, `accel_read_new()` | `Accelerometer`: the latest sample in milli-g, the data-ready flag, a sample not read before |
| `accel_range()`, `accel_set_range(r)` | the full scale, changed without touching the other bits of CTRL_REG4 |
| `accel_rate_hz()`, `accel_set_rate_hz(hz)` | the rate, and the slowest rate at or above `hz` in the present mode |
| `accel_fifo_count()`, `accel_fifo_read(dst)` | the FIFO, which is off until `fifo_begin` |
| `device_id()` | `Identifiable`: WHO_AM_I, 0x33 |
| `mode()`, `set_mode(m)` | the resolution mode |
| `fifo_begin(n)`, `fifo_end()` | turn the FIFO on in stream mode with the watermark at `n` samples, and off |
| `self_test(negative)` | the change in milli-g that the part's self-test makes |
| `reg(r)`, `set_reg(r, v)` | one register, and a write that is read back |

`begin` checks WHO_AM_I, puts the registers in a known state because the
part has no reset pin and keeps them while powered, then sets block data
update, high resolution at ±2 g, the data-ready on INT1, and 100 Hz with
the three axes on.  It answers `false` for a part that does not answer
0x33 and for a register that does not hold what was written.

A program that holds its own bus, or a test with a double of the part,
implements `Lis3dhPort` over its bus and calls the module's functions,
which take the port: `lis3dh.sample_read(port)`, `lis3dh.start(port)`.
The framing of each bus is `i2c_read_into`, `i2c_write`, `spi_read_into`
and `spi_write`, generic over `I2cBus` and `SpiBus`.

## The rules a user needs

1. Wait 5 ms after the part is powered before `begin`.  The part boots
   from its non-volatile memory in that time.
2. `begin` has to run before a reading: the part is in power-down at
   power-on.
3. A reading is converted at the range and the mode it was taken in.
   After a change of mode or rate the part settles for between one and
   seven sample periods, as the `lis3dh` crate documents, so the caller
   waits that long before it trusts a sample.
4. 1620 Hz exists in low-power mode only.  `accel_set_rate_hz` answers the
   rate taken, which is rarely the rate asked for.
5. `set_mode` clears the bit being left before it sets the new one, so
   HR and LPen are never both set.  Leaving low-power mode at 1620 Hz
   moves the rate to 1344 Hz.
6. The FIFO is off after `begin`.  `fifo_begin` puts the watermark on
   INT1 in place of the data-ready, and `fifo_end` puts the data-ready
   back.  A full FIFO reads FSS 0 with EMPTY clear, which
   `accel_fifo_count` answers as 32.
7. The self-test change is the second reading minus the first, with the
   part at ±2 g in high resolution at 100 Hz.  The sheet's Table 3 gives
   typical values of 276, 276 and 984 mg and no limits.
   `self_test_within(d, lo, hi)` compares a change with limits that the
   caller takes from the data sheet of its part's revision.
8. `device_id` answers `None` for a bus that no part drives: an SPI bus
   reads 0xFF, or 0x00 when held low.

## What is not included

- The interrupt generators for motion, free fall, 6D and 4D orientation,
  the click and double-click detection and the high-pass filter: no trait
  in devices-nv names them.  `reg` and `set_reg` reach them.
- The temperature sensor and the three auxiliary ADC inputs.  The
  sheet gives the temperature as a change per degree with no reference,
  and the auxiliary inputs are not an accelerometer.
- The SPI three-wire mode (SIM).
- The reboot of the memory content (CTRL_REG5, BOOT).

## Related packages

- [devices-nv](https://novo-lang.org/packages/devices-nv) names the
  traits this driver implements, the wiring record and the register
  helpers it uses.
- [embedded-hal-nv](https://novo-lang.org/packages/embedded-hal-nv) names
  the `I2cBus`, `SpiBus` and `GpioOut` traits and the board handles.
- [lsm303agr-nv](https://novo-lang.org/packages/lsm303agr-nv) drives the
  accelerometer and magnetometer of the LSM303AGR, a part of the same
  family at other registers.
- [lis3dsh-nv](https://novo-lang.org/packages/lis3dsh-nv) drives the
  LIS3DSH, a different register map.

## Tests

`novo test` runs the module over a double of the part that answers on an
I2C bus and on an SPI bus: the sub-address and the command byte, the
increment, WHO_AM_I, every conversion at every mode and range, the rates,
the range and mode changes, the bring-up and its refusals, the FIFO, the
self-test and the two types over the host's board handles.  The log of
bytes sent is compared for each entry point.  The suite covers every line
of `src/`.

**Hardware: NOT VERIFIED.**  See the example section.

The sources read for this driver, for their fixes and not ported, are
the ST data sheet above, Zephyr's `drivers/sensor/st/lis2dh` and the
`lis3dh` crate 0.5.0.  Each fix kept is named in `src/lis3dh.nv` where it
is kept.

## Licence

Apache-2.0.
