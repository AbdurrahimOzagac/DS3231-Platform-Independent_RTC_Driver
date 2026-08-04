# DS3231 Platform-Independent RTC Driver

A lightweight C driver for the DS3231 real-time clock, built with zero dependency on any vendor HAL. I2C transactions are injected through function pointers, so the core driver can be reused on any MCU or OS by writing a small platform "port" layer — no changes to the driver itself.

## Why this exists

Most DS3231 libraries hard-code calls like `HAL_I2C_Mem_Write()` or `Wire.write()` directly into the driver, locking it to a single platform. This driver flips that: the core (`ds3231.h` / `ds3231.c`) never calls an I2C function directly. Instead, it holds two function pointers — `i2c_write` and `i2c_read` — and calls whichever implementation was registered for the active platform.

```
application code  →  ds3231.c (device logic: registers, BCD)  →  port layer (bus I/O)  →  hardware
```

Porting to a new platform means writing one small adapter file. The driver core stays untouched.

## Repository layout

```
ds3231.h                 Public driver interface (platform-independent)
ds3231.c                 Driver implementation: register map, BCD conversion (platform-independent)
ds3231_port_stm32.h      STM32 HAL port interface
ds3231_port_stm32.c      STM32 HAL port implementation (the only file that touches HAL)
main.c                   STM32CubeIDE usage example
```

## Features

- No HAL/MCU dependency in the core driver
- Simple function-pointer based I2C abstraction (`DS3231_I2C_Write_Fn` / `DS3231_I2C_Read_Fn`)
- Supports multiple simultaneous DS3231 instances via independent handles
- Explicit success/failure return codes on every call
- Century-bit and reserved-bit masking handled internally
- Doxygen-documented public API

## Getting started (STM32 example)

```c
#include "ds3231.h"
#include "ds3231_port_stm32.h"

DS3231_Handle_t ds3231Handle;
DS3231_Time_t   ds3231Time;

/* After MX_I2C1_Init() */
DS3231_Port_STM32_Init(&ds3231Handle, &hi2c1);

/* sec, min, hour, dayOfWeek, dayOfMonth, month, year */
DS3231_Set_Time(&ds3231Handle, 0, 30, 14, 2, 5, 6, 26);

if (DS3231_Get_Time(&ds3231Handle, &ds3231Time) == 0) {
    /* ds3231Time.hour, ds3231Time.min, ... */
}
```

## Porting to a new platform

Write two files matching the naming convention `ds3231_port_<platform>.h/.c`. Each must:

1. Implement two functions matching these signatures:

   ```c
   int8_t (*)(uint8_t dev_addr, uint8_t reg_addr, const uint8_t *data, uint16_t len, void *user_ctx);  // write
   int8_t (*)(uint8_t dev_addr, uint8_t reg_addr, uint8_t *data, uint16_t len, void *user_ctx);         // read
   ```

2. Call `DS3231_Init()` with those two functions and a platform-specific context (e.g. a bus handle or file descriptor) as `user_ctx`.

`DS3231_I2C_ADDRESS` (`0x68`) is the raw 7-bit address. Platforms that expect an 8-bit address (STM32 HAL) must shift it (`<< 1`) inside the port layer; platforms that expect a 7-bit address (Arduino `Wire`, Linux `i2c-dev`) must not.

## Register map

| Address | Field |
|---------|-------|
| 0x00 | Seconds |
| 0x01 | Minutes |
| 0x02 | Hours |
| 0x03 | Day of week |
| 0x04 | Day of month |
| 0x05 | Month (bit 7 = century flag, masked internally) |
| 0x06 | Year |

All time fields are stored on-chip as BCD; the driver converts to/from decimal transparently.

## Roadmap

- [ ] Alarm 1 / Alarm 2 support
- [ ] On-chip temperature sensor readout
- [ ] SQW / 32K output pin control
- [ ] Oscillator-stop-flag (OSF) check

## License

MIT — see [LICENSE](LICENSE).
