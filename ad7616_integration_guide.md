# AD7616 Data Acquisition & SPI Transmission Guide
### Target Hardware: EVAL-AD7616SDZ Evaluation Board

---

## 1. System Overview

The **EVAL-AD7616SDZ** is an analog front-end evaluation board featuring the **AD7616**: a 16-channel, 16-bit, dual simultaneous sampling Successive Approximation Register (SAR) Analog-to-Digital Converter (ADC). 

Because the evaluation board contains only the ADC, voltage references, passive input conditioning, and power regulation, it requires an external digital controller (Microcontroller, DSP, or FPGA) to:
1. Provide clocking and conversion triggers.
2. Configure on-chip registers (gain/range, channel multiplexing, oversampling).
3. Read converted digital samples via the Serial Peripheral Interface (SPI).

```
   [Analog Inputs]
   V0A ... V7A ──┐
                 ├──► [EVAL-AD7616SDZ] ──(SPI + Control Lines)──► [Microcontroller / DSP]
   V0B ... V7B ──┘
```

---

## 2. Hardware & Jumper Configuration

Before applying power, verify the position of the configuration links (jumpers) on the EVAL-AD7616SDZ:

| Jumper Label | Setting | Description |
| :--- | :--- | :--- |
| **`SER/PAR`** | **SER** | Selects Serial (SPI) mode rather than parallel bus mode. |
| **`SER1/SER2`** | **SER1** | Configures single-line SDO output (`SDOA`). Channel A and Channel B stream sequentially. |
| **`HW/SW`** | **SW** | Enables Software Configuration mode so registers can be written via SPI. |
| **`V_DRIVE`** | **3.3V** | Sets the logic level for digital I/O lines. Must match your MCU (3.3V or 5V). |

---

## 3. Signal Interface & Pin Mapping

Connect the relevant digital lines from the board's 120-pin SDP interface connector or dedicated test points to your microcontroller:

| AD7616 Signal | Pin Direction (re: MCU) | MCU Peripheral | Description |
| :--- | :--- | :--- | :--- |
| **`RESET`** | Output from MCU | Standard GPIO | Active-low hardware reset pulse. |
| **`CONVST`** | Output from MCU | GPIO / Timer PWM | Active-low pulse initiates simultaneous conversion on A and B. |
| **`BUSY`** | Input to MCU | GPIO / External IRQ | High during conversion; falling edge indicates data is ready. |
| **`CS`** | Output from MCU | GPIO (or SPI NSS) | Active-low SPI Chip Select. |
| **`SCLK`** | Output from MCU | SPI Clock (`SCK`) | Serial clock line (supports up to $50\text{ MHz}$). |
| **`SDI`** | Output from MCU | SPI MOSI (`COPI`) | Serial data input to ADC (register writes). |
| **`SDOA`** | Input to MCU | SPI MISO (`CIPO`) | Serial data output from ADC (sample readout). |
| **`GND`** | Reference | Ground | Common reference ground between MCU and ADC board. |

---

## 4. Communication & Timing Protocol

### 4.1 SPI Bus Characteristics
* **Transfer Size:** 16-bit words.
* **SPI Mode:** Mode 1 or Mode 2 (Clock idles low/high, data latched on the appropriate edge per datasheet; typically $CPOL=0, CPHA=1$ or $CPOL=1, CPHA=1$).
* **Safe Clock Rate:** $1\text{ MHz}$ to $20\text{ MHz}$ over standard jumper wires.

### 4.2 Register Write Packet Format
Each SPI write packet consists of a 16-bit word formatted as follows:

$$\text{Bit 15: [W/}\overline{\text{R}}\text{]} \quad|\quad \text{Bits [14:9]: [Address]} \quad|\quad \text{Bits [8:0]: [Data]}$$

* **Bit 15:** Set to `1` for register write operations.
* **Bits [14:9]:** 6-bit register address.
* **Bits [8:0]:** 9-bit configuration data.

### 4.3 Conversion & Read Sequence
1. Ensure `CS` is HIGH.
2. Drive `CONVST` LOW for $\ge 25\text{ ns}$, then return it HIGH.
3. The AD7616 asserts `BUSY` HIGH while sampling and converting.
4. Wait for `BUSY` to return LOW (falling edge).
5. Drive `CS` LOW.
6. Clock out two sequential 16-bit words on `SDOA`:
   * **Word 1:** 16-bit two's complement sample from Group A.
   * **Word 2:** 16-bit two's complement sample from Group B.
7. Return `CS` HIGH.

---

## 5. Firmware Implementation (Embedded C)

### 5.1 Register Map Definitions

```c
#ifndef AD7616_H
#define AD7616_H

#include <stdint.h>
#include <stdbool.h>

/* AD7616 Register Addresses */
#define AD7616_REG_CONFIG       0x02
#define AD7616_REG_CHANNEL      0x03
#define AD7616_REG_RANGE_A1     0x04
#define AD7616_REG_RANGE_A2     0x05
#define AD7616_REG_RANGE_B1     0x06
#define AD7616_REG_RANGE_B2     0x07

/* Input Range Settings */
#define AD7616_RANGE_10V        0x00  // +/- 10V Range
#define AD7616_RANGE_5V         0x01  // +/- 5V Range
#define AD7616_RANGE_2V5        0x02  // +/- 2.5V Range

/* Hardware Abstraction Hooks (Implement using your MCU's drivers) */
void ad7616_hal_set_cs(bool level);
void ad7616_hal_set_convst(bool level);
void ad7616_hal_set_reset(bool level);
bool ad7616_hal_read_busy(void);
uint16_t ad7616_hal_spi_xfer16(uint16_t tx_data);
void ad7616_hal_delay_us(uint32_t us);
void ad7616_hal_delay_ms(uint32_t ms);

/* Driver Functions */
void ad7616_init(void);
void ad7616_write_register(uint8_t reg_addr, uint16_t reg_val);
void ad7616_read_samples(int16_t *sample_a, int16_t *sample_b);
float ad7616_raw_to_volts(int16_t raw_code, float full_scale_range);

#endif /* AD7616_H */
```

---

### 5.2 Driver Implementation

```c
#include "ad7616.h"

/**
 * @brief Transmit a 16-bit write frame to an AD7616 internal register
 */
void ad7616_write_register(uint8_t reg_addr, uint16_t reg_val) {
    // Bit 15: 1 (Write) | Bits [14:9]: Address | Bits [8:0]: Data
    uint16_t frame = (1U << 15) | ((reg_addr & 0x3F) << 9) | (reg_val & 0x01FF);
    
    ad7616_hal_set_cs(false);
    ad7616_hal_spi_xfer16(frame);
    ad7616_hal_set_cs(true);
}

/**
 * @brief Initialize device, perform hardware reset, and set default channel ranges
 */
void ad7616_init(void) {
    // 1. Establish initial pin states
    ad7616_hal_set_cs(true);
    ad7616_hal_set_convst(true);

    // 2. Hardware Reset pulse (min 50 ns pulse width)
    ad7616_hal_set_reset(false);
    ad7616_hal_delay_us(10);
    ad7616_hal_set_reset(true);
    ad7616_hal_delay_ms(15); // Wait for internal reference to stabilize

    // 3. Configure input range for Channel V0A and V0B to +/-10V
    ad7616_write_register(AD7616_REG_RANGE_A1, AD7616_RANGE_10V);
    ad7616_write_register(AD7616_REG_RANGE_B1, AD7616_RANGE_10V);

    // 4. Select Channel 0 for both Group A and Group B
    // Bits [7:4]: Group B channel select, Bits [3:0]: Group A channel select
    ad7616_write_register(AD7616_REG_CHANNEL, 0x0000);
}

/**
 * @brief Trigger conversion and read back 16-bit samples from Channel A and B
 */
void ad7616_read_samples(int16_t *sample_a, int16_t *sample_b) {
    // 1. Pulse CONVST low to latch analog inputs and trigger conversion
    ad7616_hal_set_convst(false);
    ad7616_hal_delay_us(1); // Meets minimum pulse width requirement
    ad7616_hal_set_convst(true);

    // 2. Poll until conversion is complete
    while (ad7616_hal_read_busy() == true) {
        // Alternatively, service this from an external interrupt on BUSY falling edge
    }

    // 3. Clock out conversion results over SPI
    ad7616_hal_set_cs(false);
    *sample_a = (int16_t)ad7616_hal_spi_xfer16(0x0000);
    *sample_b = (int16_t)ad7616_hal_spi_xfer16(0x0000);
    ad7616_hal_set_cs(true);
}

/**
 * @brief Convert raw signed 16-bit code to real voltage
 * @param raw_code Signed 16-bit integer from ADC
 * @param full_scale_range Range peak limit (e.g., 10.0f for +/-10V range)
 */
float ad7616_raw_to_volts(int16_t raw_code, float full_scale_range) {
    // 16-bit signed two's complement: 32768 represents full-scale range
    return ((float)raw_code * full_scale_range) / 32768.0f;
}
```

---

## 6. Mathematical Scaling & Data Conversion

The AD7616 outputs data in **two's complement 16-bit signed binary**:

$$\text{Voltage} = \frac{\text{Raw Code} \times V_{\text{RANGE}}}{32768}$$

| Input Range Selected | Binary Output ($-\text{FS}$) | Binary Output ($0\text{ V}$) | Binary Output ($+\text{FS} - 1\text{ LSB}$) |
| :--- | :--- | :--- | :--- |
| $\pm 10\text{ V}$ | `0x8000` ($-10.0\text{ V}$) | `0x0000` ($0.0\text{ V}$) | `0x7FFF` ($+9.99969\text{ V}$) |
| $\pm 5\text{ V}$ | `0x8000` ($-5.0\text{ V}$) | `0x0000` ($0.0\text{ V}$) | `0x7FFF` ($+4.99984\text{ V}$) |
| $\pm 2.5\text{ V}$ | `0x8000` ($-2.5\text{ V}$) | `0x0000` ($0.0\text{ V}$) | `0x7FFF` ($+2.49992\text{ V}$) |

---

## 7. Execution Loop Example

```c
#include "ad7616.h"
#include <stdio.h>

int main(void) {
    int16_t raw_ch_a = 0;
    int16_t raw_ch_b = 0;

    // Initialize SPI and GPIO peripherals on your MCU first
    // system_init();

    // Initialize the AD7616
    ad7616_init();

    while (1) {
        // Trigger and read conversion pair
        ad7616_read_samples(&raw_ch_a, &raw_ch_b);

        // Convert to floating-point voltages
        float volt_a = ad7616_raw_to_volts(raw_ch_a, 10.0f);
        float volt_b = ad7616_raw_to_volts(raw_ch_b, 10.0f);

        // Output data via UART / USB / CAN
        printf("CH V0A: %7.3f V | CH V0B: %7.3f V\r\n", volt_a, volt_b);

        // Sampling delay
        ad7616_hal_delay_ms(100);
    }
}
```