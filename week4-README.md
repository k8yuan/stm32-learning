# Week 4 — I2C Protocol and IMU Sensor (GY-521 / MPU-6050)

## What I Built
Wired the GY-521 to the Nucleo and streamed live accelerometer data over UART.
AX, AY, AZ values update in real time as the board is tilted.

## Wiring
```
GY-521 VCC  → Nucleo 3.3V      (NOT 5V — will damage chip)
GY-521 GND  → Nucleo GND
GY-521 SCL  → Nucleo D15 (PB8)
GY-521 SDA  → Nucleo D14 (PB9)
GY-521 AD0  → Nucleo GND       (sets address to 0x68)
XCL, XDA, INT → unconnected
```

## Key Concepts

### I2C vs SPI
| | I2C | SPI |
|--|--|--|
| Wires | 2 (SDA + SCL) | 4 (MOSI + MISO + SCK + CS) |
| Speed | Up to 400kHz | Up to 50+ MHz |
| Addressing | 7-bit address per device | Separate CS wire per device |
| Use for | Sensors, IMUs | Displays, SD cards |

### I2C Address Shift
The MPU-6050 address is 0x68. HAL requires it shifted left by 1:
```c
0x68 << 1   // HAL uses LSB for R/W direction bit internally
```

### Open-Drain — Why I2C Needs It
I2C is a shared bus. Open-drain means a pin can only pull LOW or float —
never actively drive HIGH. The pull-up resistors on the GY-521 breakout
handle HIGH. This prevents bus fights when multiple devices share the wires.

```c
GPIOB->OTYPER |= (1 << 8) | (1 << 9);  // open-drain PB8 and PB9
```

### MPU-6050 Key Registers
| Register | Address | Purpose |
|----------|---------|---------|
| PWR_MGMT_1 | 0x6B | Write 0x00 to wake chip — do this first |
| WHO_AM_I | 0x75 | Returns 0x68 — confirms chip is alive |
| ACCEL_XOUT_H | 0x3B | Start of 14-byte sensor burst |

### Reading All Axes in One Burst
```c
uint8_t buf[14];
HAL_I2C_Mem_Read(&hi2c1, 0x68<<1, 0x3B,
    I2C_MEMADD_SIZE_8BIT, buf, 14, 100);

int16_t ax = (buf[0] << 8) | buf[1];
int16_t ay = (buf[2] << 8) | buf[3];
int16_t az = (buf[4] << 8) | buf[5];
// buf[6,7] = temperature (skip)
// buf[8-13] = gyroscope XYZ

float ax_g = ax / 16384.0f;  // ±2g range
float ay_g = ay / 16384.0f;
float az_g = az / 16384.0f;
```

### What the Axes Mean
- **AX** — tilt left/right
- **AY** — tilt forward/back
- **AZ** — vertical axis. Flat on table = **1.0g** (gravity pointing straight down)
- AZ = 0 means the board is on its side

WHO_AM_I returning 0x68 confirms the chip is correctly wired and responding.
Forgetting to write 0x00 to PWR_MGMT_1 leaves the chip in sleep mode —
all reads return 0. It's essentially the chip's power switch.

## Bugs I Hit

### I2C driver files missing
Enabling HAL_I2C_MODULE_ENABLED in hal_conf.h wasn't enough —
the driver file stm32f4xx_hal_i2c.c was missing entirely.
Fix: enable I2C1 in the .ioc file and regenerate to pull in driver files.

### Duplicate handle declarations
Had huart2 and hi2c1 declared twice — once by CubeIDE, once manually.
Fix: remove manual declarations, use CubeIDE's generated ones.

### Float printing blank
snprintf with %.2f printed nothing.
Fix: add `-u _printf_float` to linker flags:
Project → Properties → C/C++ Build → Settings →
MCU GCC Linker → Miscellaneous → Other flags

### Code deleted on regeneration
Sensor reading code placed outside USER CODE blocks got wiped
when CubeIDE regenerated from the .ioc file.
Fix: always keep code inside USER CODE BEGIN / END comments.
