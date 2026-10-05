# EVAL-AD7616SDZ — Complete Guide to Acquiring ADC Data and Reading It Over SPI

**Goal:** Take analog signals into the 16 input connectors of the EVAL-AD7616SDZ, convert them with the on-board AD76116-class ADC (AD7616), and read the conversion results out through the **SPI (serial) interface** — from absolute zero to a working, streaming data acquisition system.

> ⚠️ Part naming note: the board is **EVAL-AD7616SDZ** and the ADC on it is the **AD7616** (16-channel, 16-bit, dual simultaneous sampling, SPI/QSPI/MICROWIRE/DSP compatible). Its sibling EVAL-AD7616-PSDZ carries the AD7616-P, which is **parallel-only** — SPI does not apply to the -P variant. Everything below assumes the AD7616SDZ.

---

## Table of Contents

1. [What You Are Working With (Big Picture)](#1-what-you-are-working-with-big-picture)
2. [Two Ways to Get Data Off This Board](#2-two-ways-to-get-data-off-this-board)
3. [Step 0 — Inventory & Documents You Need](#3--step-0--inventory--documents-you-need)
4. [Board Hardware Tour](#4-board-hardware-tour)
5. [The AD7616 Essentials (You Must Understand This)](#5-the-ad7616-essentials-you-must-understand-this)
6. [Path A — Zero-Code: SDP-B Controller + PC Software](#6-path-a--zero-code-sdp-b-controller--pc-software)
7. [Path B — Standalone Mode: Your Own MCU Talking SPI to the Board](#7-path-b--standalone-mode-your-own-mcu-talking-spi-to-the-board)
8. [Writing the SPI Driver (Register Map + Full Code)](#8-writing-the-spi-driver-register-map--full-code)
9. [Data Format — Converting Raw Codes to Volts](#9-data-format--converting-raw-codes-to-volts)
10. [Advanced Features (Sequencer, Burst, Oversampling, CRC)](#10-advanced-features-sequencer-burst-oversampling-crc)
11. [Throughput & Timing Budget](#11-throughput--timing-budget)
12. [Debug & Bring-Up Checklist (Zero → Final)](#12-debug--bring-up-checklist-zero--final)
13. [Common Pitfalls and How to Fix Them](#13-common-pitfalls-and-how-to-fix-them)
14. [Reference Links](#14-reference-links)

---

## 1. What You Are Working With (Big Picture)

The signal chain you are building:

```
Analog source (±2.5 V / ±5 V / ±10 V)
        │
        ▼
[J1–J4 input connectors] ──► AD7616 (16 ch, 2×SAR ADC, on-chip ref,
        │                      anti-alias filter, input clamp, 1 MΩ input Z)
        ▼
Digital interface (you choose ONE):
  • PARALLEL bus ──► EVAL-SDP-CB1Z controller ──► USB ──► PC GUI   (Path A)
  • SPI  (SCLK/CS/SDI/SDOA/SDOB) ──► YOUR MCU/FPGA                  (Path B)  ★ your goal
```

Key facts about the AD7616 (from the Analog Devices datasheet):

| Feature | Value |
|---|---|
| Channels | 16 (8× ADC "A" = V0A…V7A, 8× ADC "B" = V0B…V7B), **simultaneously sampled in pairs** (VnA with VnB) |
| Resolution | 16 bits, two's complement output |
| Input ranges | ±2.5 V, ±5 V, ±10 V (per-channel selectable in software mode) |
| Throughput | up to **2 × 1 MSPS** (1 MSPS per channel pair) |
| Analog supply | single 5 V (VCC = 4.75 V–5.25 V) |
| Digital supply (VDRIVE) | 2.3 V – 3.6 V → set by your MCU's logic level |
| Interface | parallel **or** serial (SPI/QSPI/MICROWIRE/DSP), selected by SER/PAR pin state at reset release |
| Serial data outputs | **SDOA** (ADC A results), **SDOB** (ADC B results) → 2-wire mode; or SDOA only → 1-wire mode |
| Control pins | CONVST (start conversion), BUSY (conversion in progress), RESET, REFSEL, HW_RNGSEL[1:0], SER/PAR, SER1W, CHSEL[2:0], OS[2:0], CRCEN |

The conversion handshake is always:

1. Master (your MCU) selects the channel pair (or lets the sequencer do it).
2. Master pulses **CONVST** low (≥ 25 ns pulse; ADI reference uses a ~50 ns pulse).
3. **BUSY** goes high → the dual SAR ADCs convert. In **software mode with no oversampling** this is only ~0.4 µs max.
4. Wait for **BUSY to go low**.
5. Master pulls **CS low** and clocks **16 SCLK cycles** per channel to shift the 16-bit result out of SDOA (and 16 more out of SDOB in 2-wire mode).
6. Master releases **CS high**.

---

## 2. Two Ways to Get Data Off This Board

### Path A — SDP-B controlled mode (works out of the box, PC does everything)
The board ships **default-configured for the parallel interface** with the EVAL-SDP-CB1Z controller board plugged into the 120-way connector **J10**. The PC GUI configures the ADC and captures data over USB. **No code needed**, but also **no SPI involved** — the SDP-B talks to the AD7616 over the parallel bus. Use this path first to prove your analog front-end, signals, and board are healthy.

### Path B — Standalone mode (what you asked for) ★
Remove the SDP-B, set the board links for **serial (SPI) interface**, and wire your own microcontroller (STM32, Arduino, etc.) or FPGA to the **J5 single-inline (SIL) header** plus the control-pin headers. You write a SPI driver, trigger conversions with CONVST, watch BUSY, and shift the results out of SDOA/SDOB. This is the rest of the guide.

> 🔑 **Mode selection is latched at reset:** the logic levels of `SER/PAR` (and `SER1W`, `CRCEN`, `HW_RNGSELx`, `OSx`, `BURST`) are sampled **only when RESET is released**. If you want to change interface/operating mode you must apply a **full reset** (pulse RESET low for ≥ 1.2 µs and release) with the new pin states present. Changing pins afterwards is ignored until the next full reset.

---

## 3.  Step 0 — Inventory & Documents You Need

**Hardware**
- [ ] EVAL-AD7616SDZ board (rev B or later) with the AD7616 in the 80-lead LQFP socket
- [ ] 7 V–9 V DC barrel supply (ships with the kit) → plugs into **J7**
- [ ] Signal source(s): function generator or sensors, signal within ±2.5/±5/±10 V, **never exceeding ±16.5 V abs max**
- [ ] (Path A) EVAL-SDP-CB1Z + USB cable
- [ ] (Path B) Your MCU board, e.g. STM32 Nucleo (3.3 V logic matches VDRIVE=3.3 V perfectly), jumper wires, optional logic analyzer / scope

**Documents (download and keep open)**
- [ ] UG-1012 *EVAL-AD7616SDZ/EVAL-AD7616-PSDZ User Guide* — has the **Table 3 link options** and the test-point/SIL listings
- [ ] AD7616 **data sheet** — timing diagrams (serial read, write), register descriptions, CRC polynomial
- [ ] ADI **no-OS driver** for reference C code: `github.com/analogdevicesinc/no-OS` → `drivers/adc/ad7616/ad7616.c/.h`

**Software**
- [ ] (Path A) AD7616 evaluation software (download from the AD7616 product page) + SDP-B drivers
- [ ] (Path B) STM32CubeIDE / Arduino IDE / vendor tools for your MCU

---

## 4. Board Hardware Tour

### 4.1 Connectors (Table 4 of UG-1012)

| Connector | Function |
|---|---|
| **J1** | Analog inputs V0A–V3A (screw terminal) |
| **J2** | Analog inputs V4A–V7A |
| **J3** | Analog inputs V0B–V3B |
| **J4** | Analog inputs V4B–V7B |
| **J5** | **Digital I/O SIL header — this is your SPI + control pin access in standalone mode** |
| **J6** | External reference SMA input (not needed; the on-board ADR421 2.5 V reference is used by default) |
| **J7** | Power input, 7 V–9 V DC barrel jack |
| **J8** | External VDRIVE supply input (alternative to on-board LDO) |
| **J9** | External VCC (5 V) supply input (alternative) |
| **J10** | 120-way connector to EVAL-SDP-CB1Z (Path A only) |

Each analog input has its **own GND pin right next to it** (VnAGND / VnBGND) — always connect source ground to that pin, not to a remote ground.

### 4.2 On-board power tree
`7–9 V @ J7 → ADP7104-5.0 (5 V) → ADP7118 LDOs (5 V AVDD, 3.3 V VDRIVE)`. On-board reference: **ADR421 2.5 V** fed to REFINOUT with REFSEL high (default links) → internal reference buffer produces 4.096 V on REFCAP. Nothing for you to do here unless you want an external ref via J6.

### 4.3 Link options (jumpers) — the critical step
The board has numerous 2-pin/3-pin links ( silkscreen LKx ) around the ADC. **Their factory default sets the board to parallel interface + SDP-B control + barrel-jack power.** UG-1012 **Table 3** lists every link and its default; the board also ships with a link map in the box. For **Path B (SPI, standalone, software mode)** you need these **net effects** (verify each against the silkscreen/Table 3 — link *numbers* vary by board revision, the *signal states* below are what matter):

| Signal | Required state for your task | Why |
|---|---|---|
| Power source links | Position for **VIN (J7) 7–9 V** supply | On-board regulators generate VCC + VDRIVE |
| REFSEL | **High** | Use the on-board 2.5 V reference |
| **SER/PAR** | **High** | Select **serial (SPI)** interface — latched at reset release |
| **SER1W** | **High** (2-wire) or Low (1-wire) | High = results on **both SDOA and SDOB** (required for full 1 MSPS). Low = everything interleaved on SDOA only |
| CRCEN | Low (or High if you want the CRC word) | No CRC appended after last conversion word |
| **HW_RNGSEL1, HW_RNGSEL0** | **0, 0** | **Software mode** — ranges/sequencer/OS controlled by registers over SPI (latched at reset) |
| OS2, OS1, OS0 | Any (use 0 = no oversampling to start) | Only meaningful in hardware mode; **in software mode tie/strap them to DGND** |
| CHSEL2..0 | Don't care in software mode (used in hardware mode) | Channel comes from the Channel register |
| BURST / WR | Low (burst off) to start | Burst/sequencer later |
| STBY | High (run) | Not in standby |
| VDRIVE link | 3.3 V position | Matches typical MCU I/O |

> ⚠️ **You cannot reconfigure the interface "live".** After changing SER/PAR / SER1W / HW_RNGSEL, you **must pulse RESET** so the ADC re-latches the mode pins.

### 4.4 J5 — your SPI port (standalone mode)
J5 is the single-inline header that brings the AD7616 digital pins out for debug/standalone use. The silkscreen labels next to each pin give the signal name; cross-reference with the UG-1012 test-point table. You need these nets (names per the AD7616 pinout):

| AD7616 signal | Direction (vs MCU) | MCU pin needed | Purpose |
|---|---|---|---|
| CS | MCU → ADC | SPI NSS (GPIO) | Frame select, active low |
| SCLK | MCU → ADC | SPI SCK | Serial clock |
| SDI (DB9) | MCU → ADC | SPI MOSI | Register write data in |
| **SDOA** (DB12) | ADC → MCU | SPI MISO | Conversion data, ADC A |
| **SDOB** (DB11) | ADC → MCU | **2nd SPI MISO** (or GPIO bit-bang) | Conversion data, ADC B |
| CONVST A/B | MCU → ADC | GPIO out | Start conversion (tie A and B together if separate) |
| BUSY | ADC → MCU | GPIO in | High = converting; wait for falling edge |
| RESET | MCU → ADC | GPIO out | Full reset (mode latch) |
| DGND | — | GND | **Common ground — connect first!** |
| REFSEL, SER/PAR, SER1W, HW_RNGSEL0/1 | set by links | — | Set once via jumpers (see §4.3) |

The remaining parallel-bus pins (DB0–DB15 etc.) are left unconnected/three-stated in serial mode.

---

## 5. The AD7616 Essentials (You Must Understand This)

### 5.1 Hardware mode vs software mode
- **Hardware mode** (`HW_RNGSELx ≠ 00`): channel = CHSEL pins, range = HW_RNGSEL pins, oversampling = OS pins. No SPI register writes needed — you only *read* conversion data. Simple, but inflexible.
- **Software mode** (`HW_RNGSELx = 00`): channel, per-channel ranges, oversampling, sequencer, CRC, burst — all programmed via **SPI register writes**. **This is what we use**, because you asked for full control over the input channels.

### 5.2 Conversion control
- Pulse **CONVST** (falling edge starts it; typical pulse width ≈ 50 ns — a GPIO toggle is plenty).
- **BUSY rises** and stays high during conversion (max ~400 ns SW mode, no OS). It may glitch high for ~25 ns after CONVST — ignore glitches, wait for sustained BUSY then wait for the **falling edge**.
- Only when BUSY is low are SDOA/SDOB valid.

### 5.3 Serial read timing — **the one tricky part**
From the data sheet: the **CS falling edge takes SDOA/SDOB out of three-state and presents the MSB (bit 15)** of the result. **Every rising edge of SCLK then shifts the next bit** onto the outputs.

```
CS  ‾‾‾‾\________________________________________/‾‾‾‾
SDO   <bit15><bit14><bit13> ... <bit0>
SCLK ‾‾‾‾_/‾‾‾\_/‾‾‾\_/‾‾‾\_ ... _/‾‾‾\______
         │ MSB valid immediately at CS↓
         └── each SCLK ↑ moves to the next bit → sample on SCLK ↓
```

Because the first bit appears **before** the first clock edge (non-standard "SPI"), the robust recipes are:

- **Recipe 1 (recommended, any MCU):** CS low → issue **1 dummy SCLK pulse** → then clock **16 bits**, sampling on the rising edge (CPOL=0, CPHA=0, i.e. **SPI mode 0**, 16-bit frames). The dummy pulse discards the already-consumed MSB position cleanly.
- **Recipe 2:** CS low → clock **17 bits** → keep the **lowest 16 bits** of the received word (drop the first bit).
- Do this once per channel in 2-wire mode (SDOA then SDOB), all inside a **single CS assertion** for back-to-back reads: [dummy][A15..A0][dummy][B15..B0].

> SCLK frequency: data sheet serial timing is validated well beyond 10 MHz; staying ≤ 10–20 MHz keeps you safe and still way faster than the conversion time.

### 5.4 Register write format (16-bit SPI access)
All registers are written with **one** 16-bit SPI frame:

```
bit15 = 1 (write)
bit14..bit9 = register address (6 bits)
bit8..bit0  = data
```

Register **read** = send a 16-bit frame with bit15 = 0 + address, then send **16 more clocks** to clock the register contents out of SDOA.

### 5.5 Register map (from the ADI no-OS driver)

| Addr | Name | Key fields |
|---|---|---|
| 0x02 | Configuration | bit7 SDEF (self-detect), bit6 BURSTEN, bit5 SEQEN, bits4:2 OS[2:0] (oversampling 0,2,4,8,16,32,64,128), bit1 STATUSEN, bit0 CRCEN |
| 0x03 | Channel | bits3:0 = ADC A channel (0–7), bits7:4 = ADC B channel. Default 0x00 → V0A+V0B |
| 0x04 | Range A1 | 2 bits per channel for V0A–V3A: `01`=±2.5 V, `10`=±5 V, `11`=±10 V |
| 0x05 | Range A2 | same for V4A–V7A |
| 0x06 | Range B1 | same for V0B–V3B |
| 0x07 | Range B2 | same for V4B–V7B |
| 0x20…0x27 | Sequencer stack | bit15..9 channel addr, bit8 SSREN, bits7:4 BSEL, bits3:0 ASEL — up to 32 layers |

Channel codes: 0–7 = V0..V7, 8 = VCC monitor, 9 = ALDO monitor, 12 = self-test (returns 0xAAAA on A / 0x5555 on B).

---

## 6. Path A — Zero-Code: SDP-B Controller + PC Software

Do this first to validate the board before writing any firmware.

1. **Install software first, board disconnected.** Download the AD7616 evaluation software from the AD7616 product page; run `setup.exe` (installs to `C:\Program Files\Analog Devices\AD7616\`), accept the license, then let the wizard install the **EVAL-SDP-CB1Z drivers**. Reboot if asked.
2. **Set links to factory default** (Table 3 of UG-1012): parallel interface, SDP control, J7 power, internal reference.
3. Plug the EVAL-AD7616SDZ onto the **CON A 120-way connector** of the SDP-B.
4. Connect SDP-B to the PC USB port. In Device Manager → *ADI Development Tools* → "Analog Devices System Demonstration Platform SDP-B" should appear.
5. Plug the 7–9 V supply into **J7** on the eval board.
6. Launch **AD7616** software from the Start menu. If you see a connectivity error, wait a few seconds and click **Refresh**.
7. Click **ADC Reset** to return all registers to defaults.
8. Configure: Input Range (per channel, ±2.5/±5/±10 V — must match the physical signal), Sampling Frequency (up to 1 MSPS), multiplexer/sequencer settings.
9. Connect your signal to **V0A** (and other channels), click **Sample** → Waveform tab shows the captured data; Analysis tab gives FFT/SNR/THD; Register Map tab lets you read/write every register.
10. When done: remove power (or press the SDP-B reset tact switch near the mini-USB) **before** unplugging the 120-way connector.

---

## 7. Path B — Standalone Mode: Your Own MCU Talking SPI to the Board

### 7.1 Wiring (example: STM32 Nucleo, 3.3 V logic)

| STM32 pin | → J5 net | Notes |
|---|---|---|
| GND | DGND | **Connect first, always** |
| PA4 (SPI1_NSS, GPIO) | CS | Drive as GPIO, active low |
| PA5 (SPI1_SCK) | SCLK | Mode 0, MSB first, 16-bit frames |
| PA7 (SPI1_MOSI) | SDI | Register writes |
| PA6 (SPI1_MISO) | SDOA | 16-bit RX |
| PB0 (GPIO in) | SDOB | 2nd data line — bit-banged or SPI2_MISO |
| PB1 (GPIO out) | CONVST | Pulse to start conversion |
| PB2 (GPIO in) | BUSY | Wait for falling edge |
| PB3 (GPIO out) | RESET | Full-reset pulse |

Keep wires < 15 cm; if long, series-terminate SCLK with ~33 Ω near the MCU and keep SCLK away from the analog inputs.

### 7.2 MCU peripheral configuration (STM32CubeMX)
- **SPI1**: Full-duplex master, **Mode 0** (CPOL=0, CPHA=0), MSB first, **16-bit data size**, NSS = software (GPIO), baud rate prescaler for ~10 MHz, FIFO/DCACHE handling per your part.
- **GPIOs** as above. Enable a microsecond timer (TIM or DWT CYCCNT) for the reset timing (≥ 1.2 µs) and CONVST pulse.
- **No pull-ups on MISO lines** — the AD7616 three-states SDOx when CS is high.

### 7.3 Power-up & reset sequence (do this **exactly once** at boot)
```
1. Apply 7–9 V to J7 (or J8/J9). Wait for on-board PWR_GOOD LED.
2. With SER/PAR=1, SER1W=1, HW_RNGSEL=00 already strapped by links:
3. Drive RESET low ≥ 1.2 µs, then high.  → device latches SERIAL + 2-WIRE + SOFTWARE mode.
4. Wait ≥ 10–20 µs internal POR settling.
5. (Optional sanity) Write-then-read Configuration register 0x02 and check the readback matches.
6. Program ranges (reg 0x04..0x07) and OS (reg 0x02).
7. Do ONE DUMMY CONVERSION on the chosen channel (see code) and discard it —
   after a channel change the first conversion result must be thrown away.
8. From now on: CONVST → wait BUSY↓ → clock out A and B results.
```

---

## 8. Writing the SPI Driver (Register Map + Full Code)

Complete, commented STM32 HAL implementation (works on F4/H7 with trivial changes). `SDOB` is bit-banged on a GPIO to keep SPI1 simple; if you prefer, put SDOB on SPI2-MISO instead.

```c
/* ad7616.h — register map & defines */
#ifndef AD7616_H
#define AD7616_H

#include <stdint.h>

#define AD7616_REG_CONFIG        0x02
#define AD7616_REG_CHANNEL       0x03
#define AD7616_REG_RANGE_A1      0x04   /* V0A..V3A */
#define AD7616_REG_RANGE_A2      0x05   /* V4A..V7A */
#define AD7616_REG_RANGE_B1      0x06
#define AD7616_REG_RANGE_B2      0x07

/* CONFIG bits */
#define AD7616_CFG_SDEF          (1u << 7)
#define AD7616_CFG_BURSTEN       (1u << 6)
#define AD7616_CFG_SEQEN         (1u << 5)
#define AD7616_CFG_OS(x)         (((x) & 0x7u) << 2)
#define AD7616_CFG_STATUSEN      (1u << 1)
#define AD7616_CFG_CRCEN         (1u << 0)

/* Channel register: bits[3:0]=A, bits[7:4]=B */
#define AD7616_CHA(ch)           ((ch) & 0xF)
#define AD7616_CHB(ch)           (((ch) & 0xF) << 4)

/* Range codes per 2-bit field */
#define AD7616_RANGE_2V5         1
#define AD7616_RANGE_5V          2
#define AD7616_RANGE_10V         3

/* Two's-complement raw result */
typedef struct { int16_t ch_a; int16_t ch_b; } ad7616_result_t;

int  ad7616_init(void);                                   /* reset + config */
int  ad7616_write_reg(uint8_t addr, uint16_t data);
int  ad7616_read_reg (uint8_t addr, uint16_t *data);
int  ad7616_set_range(uint8_t ch_a, uint8_t range_a,
                      uint8_t ch_b, uint8_t range_b);     /* ch = 0..7 */
int  ad7616_select_channel(uint8_t ch);                   /* 0..7, A&B together */
int  ad7616_convert(ad7616_result_t *res);                /* CONVST + SPI read */
float ad7616_to_volts(int16_t code, uint8_t range);       /* range=1/2/3 */

#endif
```

```c
/* ad7616.c — STM32 HAL implementation */
#include "ad7616.h"
#include "main.h"          /* CubeMX generated: hspi1, GPIO handles */
#include <math.h>

extern SPI_HandleTypeDef hspi1;

/* ---- GPIO abstraction (adjust port/pin names to your board) ---- */
#define CS_LOW()      HAL_GPIO_WritePin(GPIOA, GPIO_PIN_4, GPIO_PIN_RESET)
#define CS_HIGH()     HAL_GPIO_WritePin(GPIOA, GPIO_PIN_4, GPIO_PIN_SET)
#define CONVST_LOW()  HAL_GPIO_WritePin(GPIOB, GPIO_PIN_1, GPIO_PIN_RESET)
#define CONVST_HIGH() HAL_GPIO_WritePin(GPIOB, GPIO_PIN_1, GPIO_PIN_SET)
#define RESET_LOW()   HAL_GPIO_WritePin(GPIOB, GPIO_PIN_3, GPIO_PIN_RESET)
#define RESET_HIGH()  HAL_GPIO_WritePin(GPIOB, GPIO_PIN_3, GPIO_PIN_SET)
#define BUSY_READ()   HAL_GPIO_ReadPin(GPIOB, GPIO_PIN_2)
#define SDOB_READ()   HAL_GPIO_ReadPin(GPIOB, GPIO_PIN_0)

static void delay_us(uint32_t us){ /* TIM/DWT based microsecond delay */
  uint32_t start = DWT->CYCCNT;
  uint32_t ticks = us * (SystemCoreClock / 1000000U);
  while ((DWT->CYCCNT - start) < ticks) { }
}

/* ---- low level: 16-bit SPI transfer ---- */
static uint16_t spi_xfer16(uint16_t tx){
  uint16_t rx;
  HAL_SPI_TransmitReceive(&hspi1, (uint8_t*)&tx, (uint8_t*)&rx, 1, 100);
  return rx;
}

/* One SCLK pulse on SCK line (used as the "dummy" bit of the AD7616
   serial read: CS falling edge already presented bit 15). */
static void sclk_pulse(void){
  HAL_GPIO_WritePin(GPIOA, GPIO_PIN_5, GPIO_PIN_SET);
  __NOP(); __NOP();
  HAL_GPIO_WritePin(GPIOA, GPIO_PIN_5, GPIO_PIN_RESET);
  __NOP(); __NOP();
}
/* NOTE: for sclk_pulse() to work, briefly reconfigure PA5 as GPIO.
   Alternative (cleaner): use Recipe 2 below and never bit-bang SCLK. */

/* ---- AD7616 primitives ---- */

/* Full reset: pulse RESET low >= 1.2 us. Mode pins are latched on release. */
static void ad7616_reset(void){
  RESET_LOW();  delay_us(5);
  RESET_HIGH(); delay_us(20);
}

int ad7616_write_reg(uint8_t addr, uint16_t data){
  uint16_t cmd = 0x8000 | ((addr & 0x3F) << 9) | (data & 0x1FF);
  CS_LOW();  spi_xfer16(cmd);  CS_HIGH();
  return 0;
}

int ad7616_read_reg(uint8_t addr, uint16_t *data){
  uint16_t cmd = ((addr & 0x3F) << 9);   /* bit15 = 0 -> read */
  CS_LOW();
  spi_xfer16(cmd);          /* read command frame        */
  *data = spi_xfer16(0);    /* 16 dummy clocks -> data out */
  CS_HIGH();
  return 0;
}

int ad7616_set_range(uint8_t ch_a, uint8_t range_a,
                     uint8_t ch_b, uint8_t range_b){
  /* build range words: 2 bits per channel */
  uint16_t a = 0, b = 0;
  for (uint8_t ch = 0; ch < 8; ch++) {
    uint8_t ra = (ch == ch_a) ? range_a : AD7616_RANGE_10V;
    uint8_t rb = (ch == ch_b) ? range_b : AD7616_RANGE_10V;
    if (ch < 4) { a |= (ra & 3) << (ch*2); b |= (rb & 3) << (ch*2); }
    else        { a |= (ra & 3) << ((ch-4)*2); b |= (rb & 3) << ((ch-4)*2); }
  }
  ad7616_write_reg(AD7616_REG_RANGE_A1, (a >> 0) & 0xFF);
  ad7616_write_reg(AD7616_REG_RANGE_A2, (a >> 8) & 0xFF);
  ad7616_write_reg(AD7616_REG_RANGE_B1, (b >> 0) & 0xFF);
  ad7616_write_reg(AD7616_REG_RANGE_B2, (b >> 8) & 0xFF);
  return 0;
}

int ad7616_select_channel(uint8_t ch){           /* ch 0..7 */
  return ad7616_write_reg(AD7616_REG_CHANNEL, AD7616_CHA(ch) | AD7616_CHB(ch));
}

/* Recipe-2 read: 17 clocks per channel, drop the first bit. */
static uint16_t read17_drop1(GPIO_TypeDef* miso_port, uint16_t miso_pin){
  /* shift in 17 bits MSB-first on MISO, bit-banged SCLK */
  uint32_t v = 0;
  for (int i = 0; i < 17; i++) {
    HAL_GPIO_WritePin(GPIOA, GPIO_PIN_5, GPIO_PIN_SET);      /* SCLK ↑ shifts next bit */
    delay_us(0);  /* tune to your SCLK speed; or use SPI instead */
    v <<= 1;
    v |= HAL_GPIO_ReadPin(miso_port, miso_pin);
    HAL_GPIO_WritePin(GPIOA, GPIO_PIN_5, GPIO_PIN_RESET);
  }
  return (uint16_t)(v & 0xFFFF);
}

int ad7616_convert(ad7616_result_t *res){
  /* 1. start conversion */
  CONVST_LOW();  delay_us(1);  CONVST_HIGH();       /* >= 25 ns pulse */

  /* 2. wait for BUSY to fall (max ~400 ns no-OS; allow margin) */
  uint32_t timeout = 100;                            /* x 10 us safety */
  while (BUSY_READ() == GPIO_PIN_SET && --timeout) delay_us(10);
  if (!timeout) return -1;

  /* 3. read results — Recipe 2 (17 clocks, drop first bit) */
  CS_LOW();
  res->ch_a = (int16_t)read17_drop1(GPIOA, GPIO_PIN_6);   /* SDOA */
  res->ch_b = (int16_t)read17_drop1(GPIOB, GPIO_PIN_0);   /* SDOB */
  CS_HIGH();
  return 0;
}

float ad7616_to_volts(int16_t code, uint8_t range){
  float fs = (range == AD7616_RANGE_2V5) ? 2.5f :
             (range == AD7616_RANGE_5V ) ? 5.0f : 10.0f;
  return ((float)code) * (2.0f * fs) / 65536.0f;   /* two's complement */
}

int ad7616_init(void){
  /* Enable DWT cycle counter for delay_us */
  CoreDebug->DEMCR |= CoreDebug_DEMCR_TRCENA_Msk;
  DWT->CYCCNT = 0; DWT->CTRL |= DWT_CTRL_CYCCNTENA_Msk;

  CS_HIGH(); CONVST_HIGH(); RESET_HIGH();
  ad7616_reset();                       /* latches serial/2wire/software mode */

  /* no oversampling, no CRC, no status, no burst/sequencer */
  ad7616_write_reg(AD7616_REG_CONFIG, 0x0000);

  /* example: ±5 V on V0A and V0B, ±10 V elsewhere */
  ad7616_set_range(0, AD7616_RANGE_5V, 0, AD7616_RANGE_5V);

  /* select V0A/V0B and run ONE dummy conversion to flush the pipeline */
  ad7616_select_channel(0);
  ad7616_result_t dummy;
  ad7616_convert(&dummy);
  return 0;
}
```

```c
/* main.c — the acquisition loop */
ad7616_result_t res;
float v0a, v0b;

ad7616_init();

while (1) {
  if (ad7616_convert(&res) == 0) {
    v0a = ad7616_to_volts(res.ch_a, AD7616_RANGE_5V);
    v0b = ad7616_to_volts(res.ch_b, AD7616_RANGE_5V);
    /* transmit over UART/USB/CAN... e.g.:
       printf("V0A=%+8.4f V  V0B=%+8.4f V\r\n", v0a, v0b); */
  }
}
```

> **Production-grade alternative:** use ADI's official **no-OS driver** (`ad7616.c/ad7616.h` from `analogdevicesinc/no-OS`) — it contains the exact register definitions, SPI/parallel abstractions, sequencer setup, and CRC handling used on ADI reference designs. For FPGA/Zynq there is also a **SPI Engine HDL core + Linux driver** (`AD7616 Testbenches` on ADI's developer site). Both are the fastest route past hand-rolled bit-banging.

### 7.4 Minimal Arduino version (bit-banged, any 3.3 V board)

```cpp
// Pins: CS=10, SCLK=13, SDI=11, SDOA=12, SDOB=9, CONVST=8, BUSY=7, RESET=6
#define CS_LOW  digitalWrite(10, LOW)
#define CS_HIGH digitalWrite(10, HIGH)

uint16_t read17(uint8_t misoPin){
  uint32_t v = 0;
  for (int i = 0; i < 17; i++){
    digitalWrite(13, HIGH); delayMicroseconds(1);
    v = (v << 1) | digitalRead(misoPin);
    digitalWrite(13, LOW);  delayMicroseconds(1);
  }
  return v & 0xFFFF;
}

void setup(){
  for (uint8_t p : {10,13,11,8,6}) pinMode(p, OUTPUT);
  for (uint8_t p : {12,9,7})       pinMode(p, INPUT);
  CS_HIGH; digitalWrite(8, HIGH); digitalWrite(6, HIGH);
  // full reset -> latch serial 2-wire software mode (links set beforehand)
  digitalWrite(6, LOW); delayMicroseconds(5); digitalWrite(6, HIGH); delayMicroseconds(20);
  // write CONFIG = 0x0000, CHANNEL = 0x00 via SPI mode 0, MSB first
  SPI.begin(); SPI.beginTransaction(SPISettings(1000000, MSBFIRST, SPI_MODE0));
  CS_LOW; SPI.transfer16(0x8000 | (0x02 << 9) | 0x0000); CS_HIGH;  // config
  CS_LOW; SPI.transfer16(0x8000 | (0x03 << 9) | 0x0000); CS_HIGH;  // channel V0
  // dummy conversion
  digitalWrite(8, LOW); delayMicroseconds(1); digitalWrite(8, HIGH);
  while (digitalRead(7) == HIGH) { }
}

void loop(){
  digitalWrite(8, LOW); delayMicroseconds(1); digitalWrite(8, HIGH); // CONVST
  while (digitalRead(7) == HIGH) { }                                 // BUSY
  CS_LOW;
  int16_t a = (int16_t)read17(12);   // SDOA
  int16_t b = (int16_t)read17(9);    // SDOB
  CS_HIGH;
  Serial.print((float)a * 10.0f / 32768.0f); Serial.print("  ");
  Serial.println((float)b * 10.0f / 32768.0f);
}
```

---

## 9. Data Format — Converting Raw Codes to Volts

- Output is **16-bit two's complement**, MSB first.
- Full-scale mapping (no missing codes, ideal):

| Range | LSB size | Code −32768 | Code 0 | Code +32767 |
|---|---|---|---|---|
| ±10 V | 305.175 µV | −10 V | 0 V | +9.999695 V |
| ±5 V | 152.588 µV | −5 V | 0 V | +4.999847 V |
| ±2.5 V | 76.294 µV | −2.5 V | 0 V | +2.499924 V |

- Formula: `Volts = code × (2 × FS) / 65536` with `code` as a signed int16.
- Sanity check without any analog signal: select the **self-test channel (channel code 0xC)** via the Channel register → SDOA returns **0xAAAA**, SDOB returns **0x5555** (or the inverse, depending on revision). If you read those, your SPI + mode selection are correct and the digital chain is alive.

---

## 10. Advanced Features (Sequencer, Burst, Oversampling, CRC)

### 10.1 Channel sequencer + burst mode (scan all 16 channels automatically)
1. Program the **sequencer stack registers 0x20…0x27** (up to 32 layers; typically 8 layers for ch0–7): each layer word = `{addr[6:0]<<9, SSREN bit8, BSEL[3:4], ASEL[3:0]}`; set `SSREN` on the **last** layer to roll over or return.
2. Set `SEQEN=1` (and `BURSTEN=1` if you want continuous bursts) in **CONFIG (0x02)**.
3. One CONVST pulse per step (or free-run by tying CONVST low). Results stream out in layer order on SDOA/SDOB.
4. With **STATUSEN=1**, a status word is appended after the last conversion word of a sequence (bits[15:12] = last A channel, bits[11:8] = last B channel, bits[7:0] = CRC of status if CRCEN=1).

### 10.2 Oversampling
Set `OS[2:0]` in CONFIG: 0=off, 1→OSR 2, 2→4, 3→8, 4→16, 5→32, 6→64, 7→128. Higher OSR = more SNR (92 dB at OSR 2), lower max throughput, and longer BUSY — scale your BUSY timeout accordingly.

### 10.3 CRC
Strap CRCEN high at reset (or set CRCEN in software), and the ADC appends one 16-bit CRC word after the last conversion word of a sequence. Polynomial and bit order are in the data sheet's CRC section; the ADI no-OS driver contains a ready implementation. Note: **CRCEN is also latched at reset release in serial hardware mode**; in software mode it is controlled by the register.

---

## 11. Throughput & Timing Budget

Rough per-sample cost in software mode, no OS, 2-wire SPI at 10 MHz:

| Step | Time |
|---|---|
| CONVST pulse + BUSY wait | ~0.5–1 µs |
| Read 2×17 bits @ 10 MHz | ~3.4 µs |
| Software overhead | ~2–5 µs (MCU dependent) |
| **Total per pair** | **≈ 6–10 µs → 100–160 kSPS realistic on bare-metal MCU** |
| Absolute ADC limit | 1 µs per pair (hardware handshake + SPI ≥ 40 MHz, or FPGA/SPI-Engine offload) |

If you need > 500 kSPS, move the readout to an FPGA or an MCU with SPI + DMA double-buffering (e.g. STM32H7 with 16-bit frames and circular DMA), and keep SCLK ≥ 40 MHz.

---

## 12. Debug & Bring-Up Checklist (Zero → Final)

1. ☐ Links set per §4.3 → **serial, 2-wire, software mode, internal ref, J7 power**.
2. ☐ Power on, PWR_GOOD LED lit. Measure VCC ≈ 5 V, VDRIVE ≈ 3.3 V at test points.
3. ☐ J5 DGND → MCU GND connected **before any other wire**.
4. ☐ Pulse RESET, verify with scope/logic analyzer that CS/SCLK/CONVST toggle from your code.
5. ☐ `ad7616_write_reg(0x02, 0x0000)` then `ad7616_read_reg(0x02)` → readback must equal `0x0000`. If not: check SCLK/SDI wiring, mode 0, 16-bit frames, bit order.
6. ☐ Select self-test channel 0xC → read 0xAAAA / 0x5555. If not: your 17-bit/dummy-clock read alignment is off by one bit.
7. ☐ Select V0A/V0B, tie V0A to GND → code ≈ 0x0000 (± a few LSB). Tie to a known DC → code matches §9 table.
8. ☐ Apply real signal, verify in §9 units on your UART/plot.
9. ☐ Enable sequencer + burst; verify channel order with STATUSEN=1.
10. ☐ Enable OSR=2, confirm SNR improvement and longer BUSY.
11. ☐ Soak test at your target rate; watch for BUSY timeouts (→ increase timeout if OSR on; check signal not exceeding ±16.5 V).

---

## 13. Common Pitfalls and How to Fix Them

| Symptom | Cause | Fix |
|---|---|---|
| All-0 or all-1 data; no BUSY activity | Board still in parallel mode (SER/PAR latched low) | Set SER/PAR link high → **pulse RESET** |
| Data shifted by one bit (values look ×2 or /2) | The CS-falling-edge MSB consumed/not consumed | Use the 17-clock + drop-first recipe (§5.3) |
| Only one data stream | SER1W latched low (1-wire) | Set SER1W high → full reset, or parse interleaved 1-wire format |
| Readback of written register is wrong | CPHA wrong (mode 1/3), or 8-bit frames | Mode 0, **16-bit frames**, MSB first |
| Random codes when input grounded | Floating source / missing per-channel GND | Tie VnAGND/VnBGND to source ground |
| First sample after channel change is garbage | Pipeline dummy conversion | Discard one conversion after every channel-select write |
| Nothing works, MCU resets | VDRIVE ≠ MCU VDD, or ground loop | Feed VDRIVE 3.3 V, single-point ground at J5 DGND |
| Works at low rate, fails at high rate | SCLK too fast for wiring, or BUSY timeout too short | Shorten wires, terminate SCLK, increase timeout, check OSR |
| AD7616-P board (-PSDZ) | Parallel-only device | SPI impossible — use parallel or swap board |

---

## 14. Reference Links

- UG-1012 *EVAL-AD7616SDZ/EVAL-AD7616-PSDZ User Guide* (analog.com) — link options Table 3, connectors, test points
- AD7616 Data Sheet (analog.com) — serial timing, register details, CRC, sequencer
- ADI no-OS driver: `github.com/analogdevicesinc/no-OS` → `drivers/adc/ad7616/`
- ADI HDL + testbenches: `developer.analog.com` → AD7616 testbench (SPI Engine)
- ADI engineerZone thread *"AD7616 SPI interface issues"* — real-world 1-wire mode debug
- Evertiq article *"Manipulating MCU SPI interface to access a non-standard SPI ADC"* — timing workarounds for ADCs like the AD7616

---
*Prepared for standalone SPI data acquisition on the EVAL-AD7616SDZ. Always cross-check jumper numbers against the silkscreen and UG-1012 Table 3 for your board revision.*
