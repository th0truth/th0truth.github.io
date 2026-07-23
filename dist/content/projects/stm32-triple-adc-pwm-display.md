# stm32-triple-adc-pwm-display

Firmware for **STM32F103C8T6 (Blue Pill)** that reads three analog channels using **DMA**, generates **PWM** outputs, and displays sensor readings on an **ST7735 SPI TFT display** using standard C/C++ embedded drivers.

## Hardware Components
- **Microcontroller**: STM32F103C8T6 (ARM Cortex-M3)
- **Display**: ST7735 1.8" SPI TFT LCD (128x160)
- **Analog Sensors**: 3x Potentiometer / Analog Inputs (ADC1_IN0, ADC1_IN1, ADC1_IN2)
- **PWM Outputs**: TIM3 Timer Channels for PWM duty cycle regulation

## Tech Stack
- **Embedded C / C++**
- **STM32 HAL / LL Drivers**
- **ADC + DMA (Direct Memory Access)**
- **TIM PWM Generation**
- **ST7735 SPI LCD Library**
- **STM32CubeIDE / GNU Arm Embedded Toolchain**

## Key Features
- **Zero-CPU Overhead ADC Streaming**: Uses DMA to continuously transfer 3-channel ADC conversions to RAM buffers without interrupting main loop execution.
- **Dynamic PWM Control**: Maps raw 12-bit ADC readings directly to PWM duty cycles in real time.
- **Hardware SPI TFT Driver**: Optimized SPI driver for rendering live telemetry data and bar charts on the ST7735 display.
