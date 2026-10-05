# EVAL-AD7616SDZ → Standalone SPI Data Acquisition

**A complete zero-to-working guide: power the board standalone, configure it for serial (SPI) mode, drive it with your own MCU (Arduino / STM32 / Raspberry Pi), read all 16 ADC channels, and stream the data out.**

---

## 0. Read This First (Reality Check)

The EVAL-AD7616SDZ was designed primarily to run with Analog Devices' SDP controller board (EVAL-SDP-CB1Z) under the ACE software. **However, it fully supports standalone operation with your own MCU** — you must:

1. Reconfigure the jumper/link block (factory default is set up for SDP/parallel use),
2. Supply 6 V DC power yourself,
3. Connect your MCU to the AD7616's digital interface signals,
4. Write firmware that follows the AD7616 timing rules.

### ⚠️ Items you MUST verify against the latest official docs

Chip-level behavior below is solid, but **board revision details (connector numbers, link/jumper numbers, silk-screen labels) can change**. Before soldering anything, download and skim:

| # | Document | Where |
|---|----------|-------|
| 1 | **AD7616 datasheet** (pin functions, timing tables, register map) | analog.com → AD7616 → Documentation |
| 2 | **EVAL-AD7616SDZ user guide** (UG-711 at the time of writing) — contains the full board schematic, jumper table, connector list | analog.com → EVAL-AD7616SDZ → Documentation |
| 3 | **EVAL-SDP-CB1Z user guide (UG-337)** — only if you use the SDP route | analog.com |

Things to confirm in those docs (all flagged ⚠️ throughout this file):

- Exact link (LK#) numbers and default positions for: PAR/SER, H/S, OS[2:0], RANGE[2:0], VDRIVE, REFSEL, STBY
- Polarity of REFSEL and STBY
- Analog input connector type/labels on your board revision
- Exact timing limits (SCLK max, BUSY width, RESET pulse width)
- Hardware-mode sequencer behavior (§6.4)

---

## 1. What You're Working With

### 1.1 The AD7616 chip in one page

| Feature | Value |
|---|---|
| Architecture | **Two 16-bit SAR ADCs** ("A" and "B"), each fed by an 8:1 mux |
| Channels | 16 total: **V0–V7** → ADC A, **V8–V15** → ADC B |
| Simultaneity | One channel from each half converts **simultaneously** (a "pair", e.g. V0+V8) |
| Throughput | Up to ~1 MSPS on a channel pair; scanning all 8 pairs via the on-chip sequencer ≈ 1/8 of that per channel (less with oversampling) ⚠️ confirm in datasheet Table 1 |
| Input ranges | True bipolar: **±10 V, ±5 V, ±2.5 V** (selected by RANGE[2:0] pins or register) |
| Oversampling | On-chip digital filter: ×2, ×4, ×8, ×16, ×32 (OS[2:0] pins or register) |
| Code format | 16-bit **two's complement**, MSB first |
| Reference | Internal 2.5 V or external 2.5 V (REFSEL) |
| Supply | AVCC = 5 V (analog), **VDRIVE = 2.3–5.5 V** (sets I/O logic level) |
| Interface | Parallel (8/16-bit bus) **or** serial SPI (your case) |

**Key insight for your goal:** The AD7616 is an SPI *slave* with a **read-only** data port. "Transmitting ADC data over SPI" means: your MCU (SPI master) starts a conversion, waits for BUSY, then clocks the 16-bit results out of the chip's DOUT lines.

### 1.2 The evaluation board

- **Power input:** 6 V DC jack (wall adapter or bench supply). On-board regulators generate the 5 V analog rails. ⚠️ Check polarity printed on the board.
- **SDP connector:** 120-pin connector that mates with the SDP-B controller. **All the AD7616 digital interface signals (CS, SCLK, DOUTA/B, BUSY, CONVST, RESET, FRSTDATA, VDRIVE…) are routed to this connector.** For standalone use you tap these signals here (or from any auxiliary header/test points your board revision provides — check the schematic in the user guide).
- **Analog inputs:** 16 channels brought out on input connectors, labeled V0…V15 (⚠️ connector type and designators per the user guide schematic).
- **Jumper/link block:** maps every AD7616 control pin (PAR/SER, H/S, OS, RANGE, REFSEL, STBY…) to selectable levels. This is how you configure the chip **without writing any registers**.
- **Reference:** an on-board precision 2.5 V reference feeds the ADC (REFSEL link selects on-board/external vs internal).

### 1.3 System architecture we will build

```
                 +---------------------------+
  Sensor/        |   EVAL-AD7616SDZ          |
  voltage  ----->|  analog inputs V0..V15    |
  source         |                           |
                 |   AD7616 (SPI slave)      |
                 +------------+--------------+
            6 V DC power in  |
                             |  CS, SCLK, DOUTA(/DOUTB)
                             |  CONVST, BUSY, RESET, FRSTDATA
                             v
                 +----------------------+        +-----------+
                 |  Your MCU (SPI master)|--USB-->|   PC      |
                 |  STM32 / Arduino / RPi|  UART  | (viewer/  |
                 +----------------------+        |  logger)  |
                                                 +-----------+
```

---

## 2. What You Need (Bill of Materials)

| Item | Notes |
|---|---|
| EVAL-AD7616SDZ | The eval board |
| 6 V DC supply, ≥ 500 mA | 2.1 mm barrel jack typical ⚠️ verify polarity; a bench supply set to 6 V works too |
| MCU board | STM32 (Nucleo/Blue Pill) **recommended** (3.3 V logic, fast). Alternates: Raspberry Pi, Pico, ESP32, Arduino Uno (with care, §5.3) |
| Jumper wires (short!) | Dupont female-female mostly |
| USB-UART bridge | If your MCU lacks native USB (for streaming to PC) |
| Multimeter | Rail checks |
| Optional: oscilloscope | Hugely helpful for BUSY/SCLK debugging |
| Optional: test signal source | AA battery, potentiometer, function generator |
| ESD wrist strap | The ADC front end is sensitive |

**Software:** Arduino IDE *or* STM32CubeIDE *or* Python (RPi). Optionally Analog Devices ACE (only for the SDP route, §12.4).

---

## 3. Safety & ESD Rules (do not skip)

1. **Power everything OFF while rewiring.**
2. **VDRIVE defines the chip's I/O voltage.** Every MCU output pin connected to the board must not exceed VDRIVE + 0.3 V. Feeding 5 V GPIO into a board configured for VDRIVE = 3.3 V **can destroy the AD7616's digital port**. Match VDRIVE to your MCU (§5.3).
3. Analog inputs tolerate the selected range and some overdrive (they're true-bipolar inputs), but check the **Absolute Maximum Ratings** table in the datasheet before connecting anything beyond ±10 V.
4. Use a common ground: board GND ↔ MCU GND (single point).
5. ESD strap; keep the SDP connector **unpopulated** while using your own MCU.

---

## 4. Configure the Board (Jumper/Link Settings)

### 4.1 Target configuration for standalone SPI

| Function | Signal | Setting for this project | Why |
|---|---|---|---|
| Interface mode | PAR/SER/BURST | **LOW → serial (SPI)** | You want SPI |
| Mode select | H/S | **HIGH → hardware mode** | OS/range come from pins; **no register writes needed** (register writes require the parallel port) |
| Oversampling | OS[2:0] | **000** (oversample off) to start | Fastest, simplest; increase later |
| Input range | RANGE[2:0] | **000 = ±10 V** (typical encoding ⚠️ confirm table) | Full-range headroom for testing |
| Reference | REFSEL | On-board (external) reference — board default ⚠️ confirm polarity | Best accuracy |
| Standby | STBY | Normal operation (not standby) ⚠️ confirm polarity in pin table | Chip must be awake |
| I/O voltage | VDRIVE | **Match your MCU**: 3.3 V (STM32/RPi/Pico/ESP32) or 5 V (Uno) | §3 rule 2 |
| Power source | — | 6 V at the DC jack; **SDP unplugged** | Standalone |

### 4.2 How to find the actual LK numbers (minute steps)

1. Open the EVAL-AD7616SDZ user guide → section **"Jumper Link Functions"** (or equivalent) → table lists each LK# → signal it controls → positions.
2. Open the board **schematic** (last pages of the user guide) and find the link connected to the AD7616 pin named `PAR/SER/BURST`, `H/S`, `OS0..2`, `RANGE0..2`, `STBY`, `REFSEL`, `VDRIVE`.
3. Physically reposition each link per the table above. Photograph the board before/after.
4. Double-check with a multimeter (continuity mode, power off): e.g., confirm the PAR/SER/BURST pin now reads continuity to GND.

### 4.3 Power-up rail check (before connecting the MCU)

1. Connect 6 V supply, power on.
2. Measure on the board: **5 V rail** (analog), **3.3 V/5 V at VDRIVE** per your link setting.
3. If any rail is missing → stop, fix jumpers/supply first.

---

## 5. Wire the MCU to the Board

### 5.1 Signal reference (what each wire does)

| AD7616 signal | Direction (relative to board) | Function | Required? |
|---|---|---|---|
| CS | Input | Chip select, **active low**. First data bit appears on DOUT when CS falls | ✅ |
| RD/SCLK | Input | Serial clock (this pin is named RD in parallel mode) | ✅ |
| DOUTA | Output | Serial data out: ADC A result, then ADC B result over 32 clocks ⚠️ confirm order | ✅ |
| DOUTB | Output | ADC B result simultaneously with DOUTA (advanced/speed option) | optional |
| CONVSTA | Input | Conversion start, ADC A half. **Conversion starts on rising edge** (falling edge puts the track/hold in hold) | ✅ |
| CONVSTB | Input | Conversion start, ADC B half — **tie to CONVSTA** | ✅ (tied) |
| BUSY | Output | HIGH while converting. **Read only after BUSY goes LOW** | ✅ |
| RESET | Input | Digital reset — pulse HIGH ≥ ~50 ns after power-up ⚠️ confirm width/polarity | ✅ |
| FRSTDATA | Output | Pulses to mark "DOUTA now carries V0" (start-of-sequence sync) | optional (recommended) |
| WR/FS | Input | Write/Frame-sync. **Not needed for plain SPI reads** — leave at board link default (typically pulled high) ⚠️ | leave default |
| VDRIVE | Power | 3.3 V or 5 V per link | ✅ |
| GND | Power | Common ground | ✅ |

### 5.2 Wiring tables

**Arduino Uno/Nano (use VDRIVE = 5 V!):**

| Board signal | Arduino pin |
|---|---|
| SCLK (RD/SCLK) | D13 (SCK) |
| DOUTA | D12 (MISO) |
| CS | D10 (GPIO) |
| CONVSTA + CONVSTB (tied) | D9 (GPIO) |
| BUSY | D8 (GPIO input) |
| RESET | D7 (GPIO) |
| FRSTDATA | D6 (GPIO input, optional) |
| GND | GND |
| VDRIVE | set by link to 5 V (do **not** back-feed from the Uno) |

**STM32 (SPI1) — recommended (VDRIVE = 3.3 V):**

| Board signal | STM32 pin |
|---|---|
| SCLK | PA5 (SPI1_SCK) |
| DOUTA | PA6 (SPI1_MISO) |
| CS | PA4 (GPIO output) |
| CONVSTA+B | PB0 (GPIO output) |
| BUSY | PB1 (GPIO input) |
| RESET | PB2 (GPIO output) |
| FRSTDATA | PB3 (GPIO input, optional) |
| GND | GND |
| VDRIVE | link = 3.3 V |

**Raspberry Pi (VDRIVE = 3.3 V):**

| Board signal | RPi pin |
|---|---|
| SCLK | GPIO11 (SPI SCLK, physical 23) |
| DOUTA | GPIO9 (MISO, physical 21) |
| CS | GPIO8 (CE0, physical 24) |
| CONVSTA+B | GPIO23 (physical 16) |
| BUSY | GPIO24 (physical 18) |
| RESET | GPIO25 (physical 22) |
| GND | physical 6 |
| VDRIVE | link = 3.3 V |

> **If your board revision exposes these signals only on the 120-pin SDP connector:** either make/buy an SDP interposer breakout board, or carefully solder fly-wires to the schematic-identified test points/nets. Never hot-plug; identify each net with the schematic + continuity beeper.

### 5.3 Logic-level rule (print this)

```
MCU logic level  ==  VDRIVE link setting
  5 V  MCU (Uno)         →  VDRIVE = 5 V
  3.3 V MCU (STM32/RPi)  →  VDRIVE = 3.3 V
Mismatched? Use a level shifter (e.g., TXS0108E / BSS138 modules).
```

### 5.4 Wiring hygiene

- Keep SCLK/DOUTA leads < 15–20 cm for your first tests.
- One common ground point (star).
- Don't route CONVST/BUSY wires parallel to SCLK for long distances.

---

## 6. How the Readout Works (Theory of Operation)

### 6.1 Conversion cycle (hardware mode, serial read)

```
        ┌─ pulse ─┐
CONVST ─┘         └────── (idle HIGH; falling edge = track/hold freezes, rising edge = convert starts)
BUSY  ────────────┌──────┐──────────  (HIGH during conversion)
                  └──────┘
CS    ──────────┐        ┌─────────   (assert AFTER BUSY low)
SCLK            │32 edge │
DOUTA           │<A MSB..LSB><B MSB..LSB>│   (32 clocks → both results)
```

**Firmware state machine:**

```
INIT → RESET pulse → [ trigger CONVST → wait BUSY low → CS low → clock 32 bits → CS high → repeat ] → decode
```

### 6.2 SPI details

- **Data:** MSB first, 16-bit **two's complement** per result.
- **Read via DOUTA only (this guide's method):** one CS-low frame, **32 SCLKs** → first 16 bits = ADC A result (V0–V7 half), next 16 = ADC B result (V8–V15 half). ⚠️ Confirm the A-then-B order in the datasheet "Serial Interface" section.
- **Advanced (faster):** wire DOUTA *and* DOUTB to two SPI peripherals; 16 clocks each reads both results simultaneously.
- **SPI mode:** data shifts out on the falling SCLK edge; sample on the rising edge → **SPI Mode 0 (CPOL=0, CPHA=0) works; Mode 3 also works**. If your data looks bit-shifted, re-check the datasheet timing figure and try Mode 3.
- **Speed:** start at **1–2 MHz SCLK**. The interface supports much more (tens of MHz class ⚠️ check t_SCLK), but prove correctness first.
- **Do not** start a read while BUSY is high, and complete each read before triggering the next conversion.

### 6.3 Oversampling behavior

With OS[2:0] ≠ 000, each triggered "conversion" internally averages 2/4/8/16/32 samples: **BUSY stays high much longer** (tens of µs at ×32) and the returned word is the filtered average. Poll BUSY — never hard-code short delays.

### 6.4 Channel sequencing (⚠️ verify in "Hardware Mode" datasheet section)

In hardware mode the on-chip sequencer steps through all 8 pairs — each CONVST converts the next pair (V0/V8 → V1/V9 → … → V7/V15 → wraps to V0/V8). **FRSTDATA marks the frame in which DOUTA carries V0**, letting you re-sync if you ever lose count. After a RESET pulse, the sequence starts at V0/V8.

> Fallback: if your testing shows the *same* pair repeating instead of auto-advancing, your unit needs the sequencer bit via software mode (register writes over the parallel port) — see §12.1.

---

## 7. Firmware From Scratch

### 7.1 Arduino sketch (complete, hardware mode, DOUTA-only)

```cpp
/*  EVAL-AD7616SDZ standalone SPI demo — Arduino Uno/Nano
 *  Board config: PAR/SER=LOW(serial), H/S=HIGH(hw mode), OS=000, RANGE=±10V,
 *                VDRIVE=5V.  DOUTA wired to MISO; 32-clock reads.        */
#include <SPI.h>

const uint8_t PIN_CS     = 10;  // -> CS
const uint8_t PIN_CONVST = 9;   // -> CONVSTA + CONVSTB (tied)
const uint8_t PIN_BUSY   = 8;   // <- BUSY
const uint8_t PIN_RESET  = 7;   // -> RESET
const uint8_t PIN_FRST   = 6;   // <- FRSTDATA (optional)

SPISettings ad7616SPI(2000000, MSBFIRST, SPI_MODE0);

const float LSB_VOLTS = 20.0 / 65536.0;   // ±10 V range -> 305.18 uV/LSB

void ad7616Reset() {
  digitalWrite(PIN_RESET, HIGH);
  delayMicroseconds(10);
  digitalWrite(PIN_RESET, LOW);
  delay(1);
}

void startConversion() {
  digitalWrite(PIN_CONVST, LOW);          // falling edge -> track/hold
  delayMicroseconds(2);
  digitalWrite(PIN_CONVST, HIGH);         // rising edge  -> conversion
}

bool waitBusy(uint32_t timeout_ms = 20) {
  uint32_t t0 = millis();
  while (digitalRead(PIN_BUSY) == HIGH)
    if (millis() - t0 > timeout_ms) return false;   // timeout = fault
  return true;
}

/* One CS frame, 32 clocks: returns ADC-A word, fills ADC-B word */
int16_t readPair(int16_t &chB) {
  SPI.beginTransaction(ad7616SPI);
  digitalWrite(PIN_CS, LOW);
  uint16_t a = SPI.transfer16(0x0000);
  uint16_t b = SPI.transfer16(0x0000);
  digitalWrite(PIN_CS, HIGH);
  SPI.endTransaction();
  chB = (int16_t)b;
  return (int16_t)a;                       // two's complement cast!
}

void setup() {
  pinMode(PIN_CS, OUTPUT);     digitalWrite(PIN_CS, HIGH);
  pinMode(PIN_CONVST, OUTPUT); digitalWrite(PIN_CONVST, HIGH);
  pinMode(PIN_RESET, OUTPUT);  digitalWrite(PIN_RESET, LOW);
  pinMode(PIN_BUSY, INPUT);
  pinMode(PIN_FRST, INPUT);
  Serial.begin(115200);
  SPI.begin();
  ad7616Reset();
}

void loop() {
  static int16_t chA[8], chB[8];

  for (uint8_t i = 0; i < 8; i++) {        // 8 pairs = all 16 channels
    startConversion();
    if (!waitBusy()) { Serial.println("BUSY timeout!"); break; }
    chA[i] = readPair(chB[i]);
  }

  for (uint8_t i = 0; i < 8; i++) {        // pretty print
    Serial.print("V"); Serial.print(i);   Serial.print("="); Serial.print(chA[i] * LSB_VOLTS, 4);
    Serial.print("  V"); Serial.print(i + 8); Serial.print("="); Serial.println(chB[i] * LSB_VOLTS, 4);
  }
  Serial.println("----------");
  delay(500);
}
```

**Sanity check to expect:** AA battery (1.5 V) on V0 → `V0 ≈ 1.5 V` (raw code ≈ **+4915** / 0x1333) at ±10 V range.

### 7.2 STM32 (HAL) version

**CubeMX config:**
- SPI1: **Full-Duplex Master** (MOSI unused), 8-bit, MSB first, **CPOL=Low, CPHA=1 Edge (Mode 0)**, NSS = Disable, prescaler → ≈1–2 MHz.
- GPIO output: PA4 (CS, initial HIGH), PB0 (CONVST, initial HIGH), PB2 (RESET, initial LOW).
- GPIO input: PB1 (BUSY). Clock: default.

```c
/* main.c — add after generated init code */
#include <stdio.h>
extern SPI_HandleTypeDef hspi1;

#define CS_HIGH()    HAL_GPIO_WritePin(GPIOA, GPIO_PIN_4, GPIO_PIN_SET)
#define CS_LOW()     HAL_GPIO_WritePin(GPIOA, GPIO_PIN_4, GPIO_PIN_RESET)
#define CONVST_H()   HAL_GPIO_WritePin(GPIOB, GPIO_PIN_0, GPIO_PIN_SET)
#define CONVST_L()   HAL_GPIO_WritePin(GPIOB, GPIO_PIN_0, GPIO_PIN_RESET)
#define BUSY()       HAL_GPIO_ReadPin(GPIOB, GPIO_PIN_1)

static void delay_us(uint32_t us) {            /* DWT microsecond delay */
  CoreDebug->DEMCR |= CoreDebug_DEMCR_TRCENA_Msk;
  DWT->CYCCNT = 0; DWT->CTRL |= DWT_CTRL_CYCCNTENA_Msk;
  uint32_t start = DWT->CYCCNT;
  while ((DWT->CYCCNT - start) < us * (SystemCoreClock / 1000000U));
}

static void ad7616_reset(void) {
  HAL_GPIO_WritePin(GPIOB, GPIO_PIN_2, GPIO_PIN_SET);
  delay_us(10);
  HAL_GPIO_WritePin(GPIOB, GPIO_PIN_2, GPIO_PIN_RESET);
  HAL_Delay(1);
}

/* 32 clocks on DOUTA: returns A result, fills *b with B result */
static int16_t ad7616_read_pair(int16_t *b) {
  uint8_t rx[4] = {0};
  CS_LOW();
  HAL_SPI_Receive(&hspi1, rx, 4, 10);
  CS_HIGH();
  *b = (int16_t)((rx[2] << 8) | rx[3]);
  return (int16_t)((rx[0] << 8) | rx[1]);
}

static void trigger(void) { CONVST_L(); delay_us(2); CONVST_H(); }

int main(void) {
  HAL_Init(); SystemClock_Config(); MX_GPIO_Init(); MX_SPI1_Init();
  ad7616_reset();

  int16_t chA[8], chB[8];
  const float lsb = 20.0f / 65536.0f;        /* ±10 V */

  while (1) {
    for (int i = 0; i < 8; i++) {
      trigger();
      uint32_t t0 = HAL_GetTick();
      while (BUSY() == GPIO_PIN_SET)
        if (HAL_GetTick() - t0 > 20) break;   /* timeout guard */
      chA[i] = ad7616_read_pair(&chB[i]);
    }
    /* ship via UART/USB here — see §9 */
    HAL_Delay(500);
  }
}
```

### 7.3 Raspberry Pi (Python + spidev)

```python
import spidev, RPi.GPIO as GPIO, time

CS, CONVST, BUSY, RESET = 8, 23, 24, 25      # BCM numbers

spi = spidev.SpiDev(); spi.open(0, 0)
spi.mode = 0; spi.max_speed_hz = 2_000_000

GPIO.setmode(GPIO.BCM)
GPIO.setup(CONVST, GPIO.OUT, initial=GPIO.HIGH)
GPIO.setup(RESET,  GPIO.OUT, initial=GPIO.LOW)
GPIO.setup(BUSY,   GPIO.IN)

GPIO.output(RESET, True);  time.sleep(1e-5); GPIO.output(RESET, False); time.sleep(0.01)

def s16(x): return x - 65536 if x & 0x8000 else x
LSB = 20.0 / 65536.0                          # ±10 V range

def read_pair():
    GPIO.output(CONVST, False); time.sleep(2e-6); GPIO.output(CONVST, True)
    t0 = time.time()
    while GPIO.input(BUSY) and time.time() - t0 < 0.02: pass
    rx = spi.xfer2([0, 0, 0, 0])              # 32 clocks, CS(CE0) auto-managed
    return s16((rx[0] << 8) | rx[1]), s16((rx[2] << 8) | rx[3])

while True:
    vals = [read_pair() for _ in range(8)]    # 8 pairs = 16 channels
    print(["%.4f" % (a * LSB) for a, b in vals])
    time.sleep(0.5)
```

### 7.4 Uniform sampling (important for real DAQ)

Free-running `delay()` gives jittery timestamps. For a fixed sample rate:

- Best: **hardware timer interrupt** fires at f_s → ISR pulses CONVST; main loop reads when BUSY drops.
- Budget example (OS off, 2 MHz SCLK): read = 32/2 MHz = 16 µs, conversion ≈ 1 µs → ≈ 20 µs per pair → full 16-ch scan ≈ 160–200 µs → **~5–6 kScans/s** (~100 kS/s aggregate). Raise SCLK to 8–16 MHz for 4×–8× more.
- With OS ×32, per-pair time grows to tens of µs — re-budget accordingly.

---

## 8. Decoding & Sanity-Checking the Data

### 8.1 Two's complement + LSB sizes

Raw word is 16-bit two's complement. Cast to signed **before** scaling.

| Range | LSB size | Full-scale code |
|---|---|---|
| ±10 V | 305.18 µV | +32767 → +9.99985 V, −32768 → −10.0 V |
| ±5 V | 152.59 µV | ±16384 ≈ ±5 V |
| ±2.5 V | 76.29 µV | ±16384 ≈ ±2.5 V |

```c
int32_t uv = ((int32_t)code * 20000) / 65536;   /* microvolts, ±10 V range */
```

### 8.2 Bench validation ladder

| Step | Test | Expected result |
|---|---|---|
| 1 | Input open/floating | Random-ish codes near mid-region — proves link is alive but nothing more |
| 2 | GND on V0 | code ≈ 0 ± a few LSB |
| 3 | +5 V on V0 (±10 V range) | code ≈ **+16384** (0x4000) |
| 4 | 1.5 V AA battery | ≈ **+4915** |
| 5 | Battery reversed | ≈ **−4915** (confirms true-bipolar + sign handling!) |
| 6 | Function generator 1 kHz sine | Clean sine after scaling; check FRSTDATA sync if channels look swapped |
| 7 | Repeat on V8..V15 | Proves the B-side (DOUTB/second 16 clocks) path works |

---

## 9. Streaming the Data Onward

The AD7616 gives you the samples; your MCU then transmits them wherever you need:

**Option A — USB/UART to PC (simplest):** send binary frames instead of text:

```
Frame: [0xAA 0x55][seq:u16][len:u16][ch0..ch15 as int16 LE][crc16]
```
Python receiver on PC: `pyserial` → parse → `numpy`/plot. Binary framing doubles throughput vs printing floats.

**Option B — MCU-to-MCU SPI:** your MCU becomes SPI master to a second device (e.g., wireless module). Note the AD7616 never *receives* data on DOUT lines — any "transmit over SPI" happens from your MCU.

**Option C — SD card logging** (SPI SD module) for standalone capture.

Double-buffer (fill buffer A while transmitting buffer B) to avoid missing conversions.

---

## 10. Bring-Up Checklist

- [ ] All docs downloaded (datasheet + eval board user guide)
- [ ] Links set: PAR/SER=serial, H/S=hardware, OS=000, RANGE=±10 V, STBY=normal, REFSEL=board default
- [ ] VDRIVE link matches MCU logic voltage
- [ ] 6 V connected → 5 V rail present → VDRIVE correct
- [ ] SDP connector unpopulated; wiring per §5.2; common GND
- [ ] Firmware: RESET pulse present, CONVST idle HIGH, CS idle HIGH
- [ ] BUSY toggles after CONVST pulse (scope/logic analyzer if available)
- [ ] Validation ladder §8.2 passes
- [ ] All 16 channels respond (8 pairs × 2 results)

---

## 11. Troubleshooting Matrix

| Symptom | Likely cause → fix |
|---|---|
| BUSY never goes high | CONVST wiring/idle state wrong; STBY link in standby; board unpowered; RESET missing |
| BUSY stuck high | OS too deep for your timeout; no RESET at startup; CONVSTB not tied |
| Reads return 0x0000 / 0xFFFF always | CS/SCLK swapped; MISO not on DOUTA; VDRIVE off; CS never asserted |
| Values bit-shifted / nonsense patterns | Wrong SPI phase — try Mode 3; check datasheet DOUT update edge; slow SCLK to 1 MHz |
| Intermittent garbage at speed | Wires too long / SCLK too fast → shorten, slow down |
| Positive inputs read huge positives | Forgot two's-complement cast (treat as signed!) |
| Channels swapped A/B | DOUTA/DOUTB swapped; check 32-clock A-then-B order ⚠️ |
| Sequence doesn't restart at V0 | Use FRSTDATA to re-sync; RESET at scan start |
| Codes stuck at one rail | Input outside range / not connected / range link wrong |
| MCU behaves erratically or hot | **VDRIVE/logic-level mismatch** — fix immediately (§3) |
| Noisy readings | Floating inputs, long wires, ground loop, high source impedance → drive from low-Z, add OS (×4/×8) |

---

## 12. Going Further (Advanced)

### 12.1 Software mode & registers
With **H/S = LOW**, per-channel ranges, channel enables, burst mode, and sequencer options are set via the **control register** — note the AD7616's register writes are performed through the **parallel** port (there is no serial data-in pin ⚠️ confirm in the "Register Programming" section of your datasheet revision). That's why this guide uses hardware mode for MCU-only setups. Read the register map section before attempting.

### 12.2 Burst mode
Register-configured auto-retriggering of the sequencer — conversions run back-to-back without CONVST pulses. Useful for max throughput; read each result between BUSY edges.

### 12.3 Parallel interface
16-bit bus (DB0–DB15) gives the fastest readback and is required for register writes — natural fit for FPGA/timers, overkill for first bring-up.

### 12.4 Official tooling routes (no/low coding)
- **SDP-B (EVAL-SDP-CB1Z) + ACE software:** plug board onto SDP, install ACE + the AD7616 plugin, capture/export CSV. Great first sanity check that your board is healthy before custom firmware.
- **ADI no-OS drivers:** github.com/analogdevicesinc/no-OS contains an AD7616 driver (targeting embedded Linux-class platforms).
- **Linux IIO driver:** mainline Linux has an `ad7616` IIO driver (see kernel docs/bindings) for Linux-native acquisition.

---

## 13. Reference Documents

| Doc | Link hint |
|---|---|
| AD7616 datasheet | analog.com → AD7616 (PDF) |
| EVAL-AD7616SDZ user guide (schematic + link table) | analog.com → EVAL-AD7616SDZ → Documentation (UG-711 at time of writing) |
| SDP-B user guide | UG-337 |
| ACE software | analog.com → ACE |
| no-OS | github.com/analogdevicesinc/no-OS |

---

## Appendix A — Range & Oversampling Pin Encodings ⚠️ (verify in datasheet tables)

| RANGE[2:0] | Range | | OS[2:0] | Oversampling |
|---|---|---|---|---|
| 000 | ±10 V | | 000 | off |
| 001 | ±5 V | | 001 | ×2 |
| 010 | ±2.5 V | | 010 | ×4 |
| others | reserved | | 011/100/101 | ×8/×16/×32 |

## Appendix B — Definition of Done

Your project is "done" when: board powers standalone from 6 V, your MCU triggers conversions and reads all 16 channels over SPI with correct two's-complement values, a battery test reads its true voltage on any channel within a few LSB, and data streams to your PC/buffer at your target rate without BUSY timeouts.

---

*Note: where marked ⚠️, confirm against the current AD7616 datasheet and the EVAL-AD7616SDZ user guide for your board revision — Analog Devices occasionally revises jumpers, connectors, and feature availability. Everything else follows the chip's documented serial timing and is standard across published AD7616 drivers.*
