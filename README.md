# Pressure Sensor Board Firmware (STM32F103)

Firmware for a custom STM32F103 board that reads four analog pressure sensors and broadcasts the values over CAN bus. The ADC samples the sensors through DMA, the firmware converts the readings to kPa, and one CAN frame with all four values is sent every 100 ms.

The PCB is a custom design. Its design files are not published in this repository, but they can be shared on request.

## Features

- 4 analog inputs sampled with ADC1 and DMA
- Conversion of the raw readings to pressure in kPa
- CAN 2.0A frames (standard ID `0x321`) at 500 kbit/s, one frame every 100 ms
- Non-blocking transmit loop based on the system tick
- Pressure values clamped to the 16 bit range before packing, and HAL return values checked at startup

## Hardware

| Item | Value |
|------|-------|
| MCU | STM32F103C8T6 (LQFP48), 72 MHz from an 8 MHz crystal through the PLL (x9) |
| Pressure inputs | PA3, PA4, PA5, PA6 (ADC1 channels 3 to 6) |
| CAN | PA11 (RX), PA12 (TX), 500 kbit/s |
| Debug | SWD on PA13 / PA14 |
| Other | PB7 configured as a GPIO output |

## CAN frame

| Field | Value |
|-------|-------|
| ID | `0x321` (standard, 11 bit) |
| DLC | 8 |
| Period | 100 ms |

| Byte | Content |
|------|---------|
| 0-1 | Sensor 1 (PA3), unsigned 16 bit, big-endian |
| 2-3 | Sensor 2 (PA4) |
| 4-5 | Sensor 3 (PA5) |
| 6-7 | Sensor 4 (PA6) |

Each value is the pressure in units of 0.1 kPa. For example, 101.5 kPa is sent as `1015` (`0x03F7`), and a frame with `03 F7 ...` starts with a reading of 101.5 kPa.

The CAN bit timing is derived from a 36 MHz APB1 clock: prescaler 18, 1 + 2 + 1 time quanta per bit, which gives 500 kbit/s with a sample point at 75 %.

## Conversion

For each channel the firmware computes

```
v_out = adc * 5.0 / 4095
p_kPa = (v_out / 5.0 + 0.00842) / 0.002421
```

The voltage scaling in the code uses a 5 V reference, and the constants `0.00842` and `0.002421` are the offset and slope of the sensor's linear transfer function.

## Project structure

```
├── Core/
│   ├── Inc/                      # headers (main.h, HAL configuration)
│   └── Src/main.c                # application code, ADC / CAN / DMA initialization
├── cmake/                        # toolchain file and CubeMX CMake sources
├── CMakeLists.txt
├── CMakePresets.json
├── PRESSURESENSORS.ioc           # STM32CubeMX configuration (pins, clocks, CAN, ADC, DMA)
├── STM32F103XX_FLASH.ld          # linker script
└── startup_stm32f103xb.s         # startup code
```

The HAL and CMSIS drivers are not stored in the repository because they are generated from the `.ioc` file.

## Build

Requirements: STM32CubeMX with the STM32Cube FW_F1 package, the Arm GNU toolchain (`arm-none-eabi-gcc`), CMake 3.22 or newer and Ninja.

1. Open `PRESSURESENSORS.ioc` in STM32CubeMX and click **Generate Code**. This creates the `Drivers/` folder.
2. Configure and build:

```bash
cmake --preset debug
cmake --build --preset debug
```

The firmware is produced as `build/debug/PRESSURESENSORS.elf`.

## Flash and debug

Connect an ST-Link to the SWD pins (SWDIO, SWCLK, GND, 3.3 V) and flash the `.elf` with STM32CubeProgrammer, or use the STM32 extension for VS Code. To check the output, connect a CAN adapter to the bus at 500 kbit/s and look for frames with ID `0x321`.

## Related project

The [CAN bus analyzer](https://github.com/tdaska824/can__bus_analyzer) in this profile simulates frames with the same ID (`0x321`) and exports them to CSV. Decoding these pressure frames in the analyzer is a natural next step.

## Author

Theopoula Eirini Daskalou, Computer Science, Aristotle University of Thessaloniki
[LinkedIn](https://www.linkedin.com/in/thei-daskalou-3567a8358) | [GitHub](https://github.com/tdaska824)
