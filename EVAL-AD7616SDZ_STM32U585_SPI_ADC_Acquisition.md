# EVAL-AD7616SDZ → MCU SPI ADC Acquisition

## Practical firmware guide for acquiring AD7616 data and forwarding it over SPI

**Target:** Analog Devices EVAL-AD7616SDZ / AD7616\
**Primary MCU example:** STM32U585 / B-U585I-IOT02A\
**Firmware stack:** STM32CubeIDE + STM32 HAL + optional FreeRTOS\
**Interface:** AD7616 serial interface → MCU SPI master\
**Goal:** Apply an analog signal → convert it with AD7616 → acquire the
16-bit conversion result in the MCU → optionally forward the samples to
another SPI/UART/USB endpoint.

------------------------------------------------------------------------

## 1. What the EVAL-AD7616SDZ actually is

The EVAL-AD7616SDZ is an evaluation board around the **AD7616**, a
16-bit, 16-channel bipolar data-acquisition ADC.

Important architectural points:

-   16 analog input channels.
-   Inputs are organized as two simultaneous ADC groups:
    -   A: V0A ... V7A
    -   B: V0B ... V7B
-   Each channel supports independently selectable bipolar input ranges
    in software mode.
-   The AD7616 uses a 5 V analog supply.
-   Its digital I/O supply, `VDRIVE`, is 2.3 V to 3.6 V.
-   The device supports parallel and serial interfaces.
-   Serial operation is SPI/QSPI/MICROWIRE/DSP compatible.
-   Conversion is controlled by `CONVST`.
-   `BUSY` indicates conversion activity.
-   Conversion results are shifted out through `SDOA` and `SDOB`.

The evaluation board normally connects to the Analog Devices SDP
controller through its 120-pin J10 connector, but it can also be
operated standalone through the J5 header.

**For an STM32 application, standalone mode is the useful
configuration.**

------------------------------------------------------------------------

# 2. System architecture

The intended system should look like this:

``` text
                  ANALOG DOMAIN
                       |
                       v
             +-------------------+
             | EVAL-AD7616SDZ    |
             |                   |
AIN0A ------>| V0A               |
AIN0B ------>| V0B               |
AIN1A ------>| V1A               |
AIN1B ------>| V1B               |
   ...       |       AD7616      |
AIN7A ------>| V7A   16-bit      |
AIN7B ------>| V7B   16-channel  |
             +---------+---------+
                       |
              +--------+---------+
              | Digital control  |
              |                 |
MCU GPIO ---->| CONVST          |
MCU GPIO <----| BUSY            |
MCU GPIO ---->| RESET           |
              |                 |
MCU SPI SCK ->| SCLK            |
MCU SPI MOSI->| SDI             |
MCU SPI MISO<-| SDOA            |
              |                 |
              +--------+---------+
                       |
                       v
                +-------------+
                | STM32U585   |
                |             |
                | SPI Master  |
                | DMA         |
                | FreeRTOS    |
                +-------------+
                       |
                       v
             application packet
             / UART / BLE / SPI
```

The critical point is:

> **SPI does not start the ADC conversion.**

The normal acquisition sequence is:

``` text
CONVST pulse
    |
    v
ADC samples analog input
    |
    v
BUSY goes HIGH
    |
    v
conversion completes
    |
    v
BUSY goes LOW
    |
    v
MCU clocks the result through SCLK
```

------------------------------------------------------------------------

# 3. Do not confuse three different interfaces

There are three separate things involved.

## 3.1 Analog input

Example:

``` text
V0A+ / V0A-
```

The AD7616 is a bipolar ADC, so the analog input is differential.

Do not directly apply an arbitrary voltage without first checking the
selected input range and the evaluation-board input circuitry.

------------------------------------------------------------------------

## 3.2 Conversion control

``` text
CONVST
BUSY
RESET
```

These are ordinary digital control/status signals.

The MCU should normally control:

``` text
RESET
CONVST
```

and monitor:

``` text
BUSY
```

------------------------------------------------------------------------

## 3.3 Serial data interface

``` text
SCLK
SDI
SDOA
SDOB
CS
```

This is the SPI-compatible interface.

For one-wire operation you can simplify the MCU connection to:

``` text
MCU SCK   -> AD7616 SCLK
MCU MOSI  -> AD7616 SDI
MCU MISO  <- AD7616 SDOA
MCU GPIO  -> AD7616 CS
```

plus:

``` text
MCU GPIO -> CONVST
MCU GPIO -> RESET
MCU GPIO <- BUSY
GND      <-> GND
```

------------------------------------------------------------------------

# 4. Evaluation-board configuration is mandatory

This is the first major trap.

The EVAL-AD7616SDZ is **not configured for standalone serial operation
by simply connecting SPI wires**.

The board is factory-oriented toward the SDP/parallel evaluation setup.

Analog Devices documents the following standalone serial modifications:

1.  Power the board correctly.
2.  Take AD7616 out of RESET.
3.  Change `SER/PAR` to select serial operation.
4.  Configure `SER1W` if using one-wire serial.
5.  Configure hardware/software mode.
6.  Remove/avoid unwanted connections between J5 and J10 when
    appropriate.
7.  Connect the MCU to the J5 digital header.

According to Analog Devices' standalone-mode guidance:

-   `R64` should be removed.
-   `SL5` should be soldered/shorted to select serial mode.
-   `SER/PAR` must be HIGH.
-   For one-wire serial operation, `SER1W` must be LOW.
-   `LK36` and `LK37` should be placed at position C for software mode.
-   J5 provides the standalone digital interface.

Do not skip this section.

If the board remains in parallel mode, an otherwise perfect STM32 SPI
driver will not produce valid serial ADC data.

------------------------------------------------------------------------

# 5. Power configuration

The evaluation board provides onboard power circuitry.

The documented rails are approximately:

  Rail              Expected
  ---------- ---------------
  AVDD                 5.0 V
  VDRIVE       3.3 V typical
  REFINOUT             2.5 V
  REFCAP           \~4.096 V
  REGCAP            \~1.87 V
  REGCAPD           \~1.89 V

For standalone operation the board can normally be powered from the J7
input, provided the board jumpers are in the appropriate default
positions.

Alternatively, AVDD and VDRIVE can be supplied separately through their
dedicated connectors.

**Important:** VDRIVE defines the digital logic voltage. If the
STM32U585 is operating at 3.3 V, a 3.3 V VDRIVE configuration is
convenient.

Do not power the analog input from the MCU 3.3 V rail unless the signal
conditioning has been designed for it.

------------------------------------------------------------------------

# 6. J5 standalone digital interface

J5 is the important connector for MCU interfacing.

Relevant signals include:

``` text
RESET
CONVST
BUSY
SCLK
SDI
SDOA
SDOB
SER1W
SER/PAR-related configuration
configuration/status signals
GND
```

Important documented J5 references:

``` text
J5-1   RESET
J5-25  BUSY
J5-26  CONVST
```

The serial data lines are exposed through the J5 `DB` signals.

For the one-wire configuration described by Analog Devices:

``` text
DB12 / SDOA -> MCU MISO
DB10 / SDI  -> MCU MOSI
SCLK        -> MCU SCK
```

`SDOB` can be left unused for a one-wire design, depending on the exact
board configuration.

**Verify the physical J5 pin numbering against your board revision and
the current UG-1012 schematic before wiring.**

------------------------------------------------------------------------

# 7. Serial interface selection

The AD7616 selects its operating interface when it exits a full reset.

The important signal is:

``` text
SER/PAR
```

The device behaves as:

``` text
SER/PAR = 0 -> parallel
SER/PAR = 1 -> serial
```

Therefore:

``` text
RESET LOW
      |
      | configuration pins valid
      v
RESET HIGH
      |
      v
device latches interface configuration
```

Changing `SER/PAR` after the device is already running does not
dynamically change the interface.

A **full reset is required** when changing this configuration.

------------------------------------------------------------------------

# 8. One-wire vs two-wire serial mode

The AD7616 supports serial one-wire and two-wire configurations.

## One-wire

Use:

``` text
SDOA -> MCU MISO
```

The MCU clocks the conversion results sequentially.

For a channel pair:

``` text
16 clocks -> A result
16 clocks -> B result
```

Therefore:

``` text
32 SCLK pulses
```

are required to retrieve both results.

Typical sequence:

``` text
CONVST
  |
BUSY
  |
BUSY falling
  |
CS LOW
  |
16 clocks -> VxA
  |
16 clocks -> VxB
  |
CS HIGH
```

------------------------------------------------------------------------

## Two-wire

The two ADC groups can be shifted simultaneously using:

``` text
SDOA -> MCU MISO-A
SDOB -> MCU MISO-B
```

This reduces the serial transfer time.

For an initial STM32 implementation, **one-wire mode is easier to
debug**.

Once the basic driver is working, move to two-wire if the sampling
throughput requires it.

------------------------------------------------------------------------

# 9. Software mode vs hardware mode

This distinction is critical.

## Hardware mode

Configuration is primarily through hardware pins.

Advantages:

-   Simple.
-   Good for fixed production configurations.
-   No register programming required for many settings.

Disadvantages:

-   Less flexible.
-   Register access is restricted.

------------------------------------------------------------------------

## Software mode

Configuration is performed through AD7616 registers.

This is the recommended mode for firmware development because the MCU
can configure:

-   input ranges
-   channel selection
-   oversampling
-   sequencer
-   burst operation
-   CRC
-   interface configuration features

The evaluation-board standalone procedure recommends putting the board
into software mode using the hardware configuration pins.

------------------------------------------------------------------------

# 10. Recommended development configuration

For your first STM32U585 implementation, use:

``` text
Interface:       Serial
Serial mode:     1-wire
Operating mode:  Software
Input range:     ±10 V initially
Oversampling:    OFF
Sequencer:       OFF initially
CRC:             OFF initially
SPI:             Master
Bit order:       MSB first
```

Why ±10 V initially?

Because it provides the largest input headroom and makes early
wiring/debugging less likely to violate the ADC input range.

After the acquisition path is working, select ±5 V or ±2.5 V where
appropriate for better LSB resolution.

------------------------------------------------------------------------

# 11. AD7616 conversion sequence

The firmware should be designed as a state machine.

``` text
             +----------------+
             |      IDLE      |
             +-------+--------+
                     |
                     v
             +----------------+
             | Assert CONVST  |
             +-------+--------+
                     |
                     v
             +----------------+
             | Wait BUSY HIGH |
             +-------+--------+
                     |
                     v
             +----------------+
             | Wait BUSY LOW  |
             +-------+--------+
                     |
                     v
             +----------------+
             | SPI read 16bit |
             +-------+--------+
                     |
                     v
             +----------------+
             | Store sample   |
             +-------+--------+
                     |
                     v
             +----------------+
             | Next channel / |
             | next trigger   |
             +----------------+
```

Do not use a large arbitrary delay in the final implementation.

Use `BUSY` as the hardware synchronization point.

------------------------------------------------------------------------

# 12. RESET sequence

RESET is active low.

A robust startup sequence is:

``` c
RESET = 0;
delay_us(...);
RESET = 1;
delay_ms(15);
```

The ADI no-OS driver uses a reset low pulse and then allows
approximately 15 ms for the device to completely reconfigure.

For an initial bring-up implementation:

``` c
HAL_GPIO_WritePin(AD7616_RESET_GPIO_Port,
                  AD7616_RESET_Pin,
                  GPIO_PIN_RESET);

HAL_Delay(2);

HAL_GPIO_WritePin(AD7616_RESET_GPIO_Port,
                  AD7616_RESET_Pin,
                  GPIO_PIN_SET);

HAL_Delay(15);
```

For production firmware, replace millisecond blocking delays with a
startup state machine or a controlled initialization delay if required
by the RTOS architecture.

------------------------------------------------------------------------

# 13. SPI electrical configuration

The MCU is the SPI master.

Recommended initial settings:

``` text
Direction:       Full duplex
Master:          Yes
Bit order:       MSB first
Frame size:      16 bits
NSS:             Software GPIO
DMA:             Enabled later
```

The AD7616 timing diagrams should be treated as authoritative for clock
polarity/phase.

For STM32 implementations, **SPI mode 3 is the safe starting
configuration**:

``` text
CPOL = 1
CPHA = 1
```

That corresponds to an idle-high SCLK.

Analog Devices' current no-OS implementation and Linux device binding
are also useful references when validating SPI timing.

Always verify with a logic analyzer/oscilloscope rather than assuming a
mode based only on an MCU SPI-mode number.

------------------------------------------------------------------------

# 14. STM32U585 CubeMX configuration

Create an SPI peripheral, for example:

``` text
SPI1
```

Configure:

``` text
Mode:
    Full-Duplex Master

Frame:
    16-bit

Clock:
    Start conservatively, e.g. 1 MHz

First Bit:
    MSB First

Clock Polarity:
    High

Clock Phase:
    2nd Edge

NSS:
    Software

CRC:
    Disabled initially
```

Use GPIOs for:

``` text
AD7616_CS
AD7616_RESET
AD7616_CONVST
AD7616_BUSY
```

Recommended:

``` text
CS      -> GPIO output
RESET   -> GPIO output
CONVST  -> GPIO output
BUSY    -> GPIO input / EXTI
```

------------------------------------------------------------------------

# 15. Example STM32 signal mapping

Example only; choose actual pins in CubeMX based on your board design.

``` text
STM32U585                    EVAL-AD7616SDZ

SPI1_SCK   ----------------> SCLK
SPI1_MOSI  ----------------> SDI
SPI1_MISO  <---------------- SDOA

GPIO_OUT   ----------------> CS
GPIO_OUT   ----------------> RESET
GPIO_OUT   ----------------> CONVST
GPIO_IN    <---------------- BUSY

GND        ------------------ GND
```

Do not connect a 5 V logic signal directly to an STM32 GPIO.

The AD7616 digital side should be operated with an appropriate VDRIVE
level, normally 3.3 V for a 3.3 V MCU system.

------------------------------------------------------------------------

# 16. Basic polling driver

Before DMA and FreeRTOS, prove the electrical interface with a blocking
driver.

``` c
#include "ad7616.h"

static HAL_StatusTypeDef AD7616_WaitBusyLow(uint32_t timeout_ms)
{
    uint32_t start = HAL_GetTick();

    while (HAL_GPIO_ReadPin(AD7616_BUSY_GPIO_Port,
                            AD7616_BUSY_Pin) == GPIO_PIN_SET)
    {
        if ((HAL_GetTick() - start) >= timeout_ms)
        {
            return HAL_TIMEOUT;
        }
    }

    return HAL_OK;
}

HAL_StatusTypeDef AD7616_ConvertRead(uint16_t *sample_a,
                                     uint16_t *sample_b)
{
    uint16_t tx = 0x0000;
    uint16_t rx_a = 0;
    uint16_t rx_b = 0;

    if ((sample_a == NULL) || (sample_b == NULL))
    {
        return HAL_ERROR;
    }

    /* Start conversion. */
    HAL_GPIO_WritePin(AD7616_CONVST_GPIO_Port,
                      AD7616_CONVST_Pin,
                      GPIO_PIN_SET);

    /* Ensure a real rising edge is generated. */
    __NOP();
    __NOP();

    HAL_GPIO_WritePin(AD7616_CONVST_GPIO_Port,
                      AD7616_CONVST_Pin,
                      GPIO_PIN_RESET);

    /* Wait for conversion completion. */
    if (AD7616_WaitBusyLow(10) != HAL_OK)
    {
        return HAL_TIMEOUT;
    }

    HAL_GPIO_WritePin(AD7616_CS_GPIO_Port,
                      AD7616_CS_Pin,
                      GPIO_PIN_RESET);

    /* First 16 clocks -> channel A. */
    if (HAL_SPI_TransmitReceive(&hspi1,
                                (uint8_t *)&tx,
                                (uint8_t *)&rx_a,
                                1,
                                10) != HAL_OK)
    {
        HAL_GPIO_WritePin(AD7616_CS_GPIO_Port,
                          AD7616_CS_Pin,
                          GPIO_PIN_SET);

        return HAL_ERROR;
    }

    /* Second 16 clocks -> channel B. */
    if (HAL_SPI_TransmitReceive(&hspi1,
                                (uint8_t *)&tx,
                                (uint8_t *)&rx_b,
                                1,
                                10) != HAL_OK)
    {
        HAL_GPIO_WritePin(AD7616_CS_GPIO_Port,
                          AD7616_CS_Pin,
                          GPIO_PIN_SET);

        return HAL_ERROR;
    }

    HAL_GPIO_WritePin(AD7616_CS_GPIO_Port,
                      AD7616_CS_Pin,
                      GPIO_PIN_SET);

    *sample_a = rx_a;
    *sample_b = rx_b;

    return HAL_OK;
}
```

**Important:** verify the STM32 SPI data register/byte order
experimentally. A 16-bit SPI peripheral normally handles the MSB-first
transfer correctly, but the received memory representation can depend on
the HAL configuration.

------------------------------------------------------------------------

# 17. Better implementation: DMA

For your actual system, DMA is preferable.

The CPU should not spend its time manually clocking every ADC sample.

Architecture:

``` text
CONVST
   |
   v
AD7616 conversion
   |
BUSY falling
   |
   v
SPI DMA
   |
   +----> ADC buffer
             |
             v
       processing task
             |
             v
       application packet
```

For two 16-bit results:

``` c
uint16_t adc_rx[2];
uint16_t adc_tx[2] = {0x0000, 0x0000};
```

Then:

``` c
HAL_SPI_TransmitReceive_DMA(&hspi1,
                            (uint8_t *)adc_tx,
                            (uint8_t *)adc_rx,
                            2);
```

Do not release CS until both 16-bit words have been clocked.

------------------------------------------------------------------------

# 18. DMA callback design

Do not perform heavy processing inside the SPI DMA ISR.

Bad:

``` c
void HAL_SPI_TxRxCpltCallback(SPI_HandleTypeDef *hspi)
{
    process_everything();
    format_large_packet();
    printf(...);
    HAL_UART_Transmit(...);
}
```

Better:

``` c
volatile bool ad7616_dma_done;

void HAL_SPI_TxRxCpltCallback(SPI_HandleTypeDef *hspi)
{
    if (hspi != &hspi1)
    {
        return;
    }

    ad7616_dma_done = true;
}
```

For FreeRTOS:

``` c
BaseType_t higher_priority_task_woken = pdFALSE;

void HAL_SPI_TxRxCpltCallback(SPI_HandleTypeDef *hspi)
{
    if (hspi != &hspi1)
    {
        return;
    }

    vTaskNotifyGiveFromISR(adcTaskHandle,
                           &higher_priority_task_woken);

    portYIELD_FROM_ISR(higher_priority_task_woken);
}
```

The ADC task then processes the buffer.

------------------------------------------------------------------------

# 19. FreeRTOS architecture

For your STM32U585 project, use separate responsibilities.

``` text
+----------------------------+
| AD7616 Acquisition Task    |
|                            |
| CONVST                     |
| BUSY synchronization       |
| SPI DMA                    |
| buffer management          |
+-------------+--------------+
              |
              v
+----------------------------+
| ADC Processing Task        |
|                            |
| sign conversion            |
| voltage conversion         |
| filtering                  |
| calibration                |
+-------------+--------------+
              |
              v
+----------------------------+
| Communication Task         |
|                            |
| packet generation          |
| CRC                        |
| UART/BLE/SPI transmission  |
+----------------------------+
```

This is much safer than placing the entire ADC/SPI/packet system inside
one FreeRTOS task.

------------------------------------------------------------------------

# 20. ADC acquisition task example

Conceptual architecture:

``` c
static uint16_t adc_dma_rx[2];
static uint16_t adc_dma_tx[2] = {0x0000, 0x0000};

void StartAdcTask(void *argument)
{
    for (;;)
    {
        /* Start conversion. */
        HAL_GPIO_WritePin(AD7616_CONVST_GPIO_Port,
                          AD7616_CONVST_Pin,
                          GPIO_PIN_SET);

        /* Minimum pulse timing must satisfy AD7616 timing. */
        __NOP();
        __NOP();

        HAL_GPIO_WritePin(AD7616_CONVST_GPIO_Port,
                          AD7616_CONVST_Pin,
                          GPIO_PIN_RESET);

        /* Wait for BUSY to become inactive. */
        while (HAL_GPIO_ReadPin(AD7616_BUSY_GPIO_Port,
                                AD7616_BUSY_Pin) == GPIO_PIN_SET)
        {
            ulTaskNotifyTake(pdTRUE, pdMS_TO_TICKS(10));
        }

        HAL_GPIO_WritePin(AD7616_CS_GPIO_Port,
                          AD7616_CS_Pin,
                          GPIO_PIN_RESET);

        if (HAL_SPI_TransmitReceive_DMA(&hspi1,
                                        (uint8_t *)adc_dma_tx,
                                        (uint8_t *)adc_dma_rx,
                                        2) != HAL_OK)
        {
            HAL_GPIO_WritePin(AD7616_CS_GPIO_Port,
                              AD7616_CS_Pin,
                              GPIO_PIN_SET);

            /* Report/recover from SPI failure. */
            continue;
        }

        ulTaskNotifyTake(pdTRUE, pdMS_TO_TICKS(10));

        HAL_GPIO_WritePin(AD7616_CS_GPIO_Port,
                          AD7616_CS_Pin,
                          GPIO_PIN_SET);

        /* adc_dma_rx[0] = channel A */
        /* adc_dma_rx[1] = channel B */

        /* Queue samples for processing. */
    }
}
```

For production code, use a proper BUSY EXTI notification rather than
polling.

------------------------------------------------------------------------

# 21. Recommended BUSY interrupt architecture

The best sequence is:

``` text
CONVST rising
       |
       v
BUSY rising
       |
       |
conversion
       |
       v
BUSY falling
       |
       v
EXTI ISR
       |
       v
ADC task notification
       |
       v
SPI DMA
```

This eliminates unnecessary CPU polling.

The ISR should only:

1.  identify the BUSY transition;
2.  notify the ADC task;
3.  return.

------------------------------------------------------------------------

# 22. SPI DMA + CS timing

This deserves special attention.

You need:

``` text
CS LOW
SCLK x 16
SCLK x 16
CS HIGH
```

not:

``` text
CS LOW
SCLK x 16
CS HIGH

CS LOW
SCLK x 16
CS HIGH
```

unless the selected AD7616 operating mode explicitly permits that
transaction structure.

For normal one-wire conversion-data acquisition, keep CS asserted for
the complete read sequence.

------------------------------------------------------------------------

# 23. Reading one channel pair

Assume the conversion selects:

``` text
V0A
V0B
```

After `CONVST` and `BUSY` completion:

``` text
CS       ______________________________
          \                            /
SCLK      _|-|_|-|_|-|_ ... _|-|_|-|_
              16 clocks     16 clocks

SDOA      <---- V0A --------><---- V0B ---->
```

The MCU receives:

``` c
adc[0] = V0A
adc[1] = V0B
```

For a selected channel pair `VnA/VnB`:

``` c
adc[0] = VnA
adc[1] = VnB
```

------------------------------------------------------------------------

# 24. Channel sequencing

The AD7616 has a flexible channel sequencer.

For initial development, do not start with the sequencer.

Use:

``` text
single conversion
single selected channel pair
manual SPI read
```

First prove:

``` text
V0A
V0B
```

Then move to:

``` text
V1A
V1B
...
```

Finally configure the sequencer.

The ADI documentation shows that the sequencer can be programmed using
registers such as the sequence stack addresses.

Example sequence configuration reported by Analog Devices for a full
8-pair sequence includes:

``` text
0x20 -> V0A/V0B
0x21 -> V1A/V1B
0x22 -> V2A/V2B
...
0x27 -> V7A/V7B
```

with the final entry marked as the sequence terminator.

------------------------------------------------------------------------

# 25. Software register access

In software mode, the AD7616 exposes configuration registers through the
serial interface.

A register write is not identical to a conversion-data read.

A common write transaction is:

``` text
CS LOW
16-bit command/data
additional clocks as required
CS HIGH
```

Do not assume that a generic `HAL_SPI_Transmit()` call with one 16-bit
word is sufficient for every AD7616 register operation.

An Analog Devices EngineerZone report demonstrates a practical issue
where a register write required the complete 32 clock cycles under the
same CS assertion.

Use the AD7616 datasheet timing diagrams as the authoritative protocol
definition.

------------------------------------------------------------------------

# 26. Useful AD7616 register commands

A commonly used interface-check command is:

``` text
0x86BB
```

This is useful during initial SPI bring-up.

Analog Devices documents that after enabling the interface-check
feature, subsequent ADC reads can return:

``` text
Channel A -> 0xAAAA
Channel B -> 0x5555
```

This is extremely useful because it separates:

``` text
SPI wiring/protocol problem
```

from:

``` text
analog-input problem
```

Recommended bring-up order:

``` text
1. Power
2. RESET
3. Serial interface configuration
4. SPI clock
5. Interface-check test
6. ADC conversion
7. Real analog signal
```

------------------------------------------------------------------------

# 27. Converting the ADC code to voltage

The AD7616 uses bipolar two's-complement coding.

For a ±10 V range:

``` text
-10 V -> approximately 0x8000
  0 V -> approximately 0x0000
+10 V -> approximately 0x7FFF
```

The ideal scale is approximately:

``` text
VLSB = 20 V / 65536
     = 305.17578125 µV
```

For signed code:

``` c
int16_t code = (int16_t)raw;

float voltage =
    ((float)code * 20.0f) / 65536.0f;
```

For ±5 V:

``` c
float voltage =
    ((float)code * 10.0f) / 65536.0f;
```

For ±2.5 V:

``` c
float voltage =
    ((float)code * 5.0f) / 65536.0f;
```

For production firmware, avoid floating-point conversion in the
acquisition path unless required.

Use fixed-point arithmetic if CPU determinism matters.

------------------------------------------------------------------------

# 28. Fixed-point conversion

For ±10 V:

``` text
full-scale span = 20 V
```

Represent voltage in microvolts:

``` c
int32_t voltage_uV =
    ((int32_t)code * 20000000LL) / 65536;
```

For ±5 V:

``` c
int32_t voltage_uV =
    ((int32_t)code * 10000000LL) / 65536;
```

For ±2.5 V:

``` c
int32_t voltage_uV =
    ((int32_t)code * 5000000LL) / 65536;
```

Use 64-bit intermediate arithmetic to prevent overflow.

------------------------------------------------------------------------

# 29. Do not use floating-point ADC storage unless necessary

Instead of:

``` c
float adc_voltage;
```

prefer:

``` c
typedef struct
{
    int16_t raw;
    int32_t voltage_uV;
} AD7616Sample_t;
```

or, for a high-throughput stream:

``` c
typedef struct
{
    uint32_t sequence;
    uint32_t timestamp_us;
    int16_t channel_a;
    int16_t channel_b;
} AD7616SamplePair_t;
```

This preserves the raw ADC data.

You can convert it later.

------------------------------------------------------------------------

# 30. DMA buffer design

For multiple channel pairs:

``` c
#define AD7616_PAIR_COUNT 8U

static uint16_t adc_rx[AD7616_PAIR_COUNT * 2U];
```

Expected ordering:

``` text
adc_rx[0] -> V0A
adc_rx[1] -> V0B

adc_rx[2] -> V1A
adc_rx[3] -> V1B

adc_rx[4] -> V2A
adc_rx[5] -> V2B

...
```

For continuous acquisition, use double buffering:

``` text
DMA buffer A
       |
       v
CPU processes A

DMA buffer B
       |
       v
CPU processes B
```

This avoids stopping acquisition while processing data.

------------------------------------------------------------------------

# 31. FreeRTOS queue

A clean design is:

``` c
typedef struct
{
    uint32_t timestamp;
    uint32_t sequence;

    int16_t channel_a;
    int16_t channel_b;

} AD7616Sample_t;
```

Then:

``` text
ISR
 |
 v
DMA complete
 |
 v
ADC task
 |
 v
xQueueSend()
 |
 v
processing task
```

Avoid dynamically allocating each ADC sample.

Use a statically allocated queue.

------------------------------------------------------------------------

# 32. If you need to forward ADC data through another SPI

There are two separate SPI roles.

## SPI #1

``` text
STM32 master
      |
      v
EVAL-AD7616SDZ
```

ADC acquisition.

## SPI #2

``` text
STM32 master
      |
      v
external controller
```

Data forwarding.

Do not attempt to share the same SPI peripheral between acquisition and
forwarding unless there is a strong reason.

For a real-time acquisition system:

``` text
SPI1 -> AD7616
SPI2 -> external controller
```

is cleaner.

------------------------------------------------------------------------

# 33. Recommended packet structure for forwarding

For your existing telemetry architecture, use something like:

``` text
+--------+------+-----+------+----------+---------+------+
| START  | TYPE | VER | SEQ  | LEN      | PAYLOAD | CRC  |
+--------+------+-----+------+----------+---------+------+
| 0xAA   | 1B   | 1B  | 2B   | 2B       | N bytes | 4B   |
```

Payload:

``` c
typedef struct __attribute__((packed))
{
    uint32_t timestamp_us;

    int16_t  channel_a;
    int16_t  channel_b;

    uint16_t channel_a_mV;
    uint16_t channel_b_mV;

} AD7616Telemetry_t;
```

Keep the raw 16-bit ADC code in the packet if possible.

------------------------------------------------------------------------

# 34. CRC strategy

There are two independent CRC problems:

### AD7616 CRC

The ADC supports CRC functionality for its serial data path.

### Application packet CRC

Your STM32-to-controller packet can have a separate CRC32.

Do not confuse them.

Recommended development sequence:

``` text
Phase 1:
AD7616 CRC disabled

Phase 2:
ADC acquisition verified

Phase 3:
AD7616 CRC enabled

Phase 4:
application packet CRC32 enabled
```

This isolates failures.

------------------------------------------------------------------------

# 35. Recommended STM32 software architecture

Use:

``` text
Core/
├── Inc/
│   ├── ad7616.h
│   ├── adc_acquisition.h
│   ├── adc_processing.h
│   └── adc_protocol.h
│
└── Src/
    ├── ad7616.c
    ├── adc_acquisition.c
    ├── adc_processing.c
    └── adc_protocol.c
```

### `ad7616.c`

Owns:

``` text
RESET
register access
channel selection
range configuration
conversion read
status
```

### `adc_acquisition.c`

Owns:

``` text
CONVST
BUSY synchronization
SPI DMA
buffer management
```

### `adc_processing.c`

Owns:

``` text
signed conversion
calibration
filtering
scaling
range conversion
```

### `adc_protocol.c`

Owns:

``` text
telemetry packet
sequence number
CRC
serialization
```

This separation is especially useful with FreeRTOS.

------------------------------------------------------------------------

# 36. Initial bring-up test

Do not start with your final application.

Use a dedicated test firmware.

## Test 1 --- Power

Verify:

``` text
AVDD ≈ 5 V
VDRIVE ≈ 3.3 V
REFINOUT ≈ 2.5 V
```

------------------------------------------------------------------------

## Test 2 --- RESET

Scope:

``` text
RESET
```

Expected:

``` text
LOW
|
+---- reset pulse
|
HIGH
```

Then allow the device setup time.

------------------------------------------------------------------------

## Test 3 --- CONVST/BUSY

Scope:

``` text
CONVST
BUSY
```

Expected:

``` text
CONVST  ____|‾|________

BUSY    ____|‾‾‾‾|____
```

BUSY must respond to conversion start.

------------------------------------------------------------------------

## Test 4 --- SPI clock

Scope:

``` text
CS
SCLK
MISO
MOSI
```

Check:

``` text
CS LOW
SCLK activity
MOSI valid
MISO valid
CS HIGH
```

------------------------------------------------------------------------

## Test 5 --- Interface check

Write:

``` text
0x86BB
```

Then perform conversion/read.

Expected test values:

``` text
A = 0xAAAA
B = 0x5555
```

If this fails, **do not debug the analog signal yet**.

------------------------------------------------------------------------

# 37. Logic-analyzer debugging

Capture at least:

``` text
CS
SCLK
MOSI
MISO
CONVST
BUSY
```

The minimum useful capture is:

``` text
CONVST
BUSY
CS
SCLK
MISO
```

Ideal timing:

``` text
             conversion             read

CONVST  ____|‾|____________________________

BUSY    ____|‾‾‾‾‾‾‾|_____________________

CS      __________________|‾‾‾‾‾‾‾|_______
                         LOW

SCLK    __________________|_|-|_|-|_|-|____

MISO    __________________< ADC DATA >_____
```

------------------------------------------------------------------------

# 38. Common failure modes

## Symptom: MISO always 0x0000

Check:

``` text
SER/PAR
SER1W
RESET
VDRIVE
SDOA connection
CS
SCLK
CONVST
BUSY
```

Also check that the board is actually in serial mode.

------------------------------------------------------------------------

## Symptom: MISO is 0xFFFF

Possible causes:

``` text
SDOA floating
incorrect VDRIVE
incorrect CS
incorrect SPI mode
wrong board configuration
```

------------------------------------------------------------------------

## Symptom: BUSY never changes

Check:

``` text
AVDD
reference
RESET
CONVST
GND
```

If BUSY does not respond, SPI debugging is premature.

------------------------------------------------------------------------

## Symptom: BUSY works but MISO is dead

Check:

``` text
SER/PAR = HIGH
SDOA path
CS
SCLK
SER1W
J5/J10 resistor conflicts
```

------------------------------------------------------------------------

## Symptom: Interface check returns wrong pattern

Check:

``` text
SPI CPOL/CPHA
MSB-first
16-bit frame
CS duration
SCLK frequency
SDI
```

------------------------------------------------------------------------

## Symptom: register reads work but writes don't

Check the complete register-write transaction.

A write may require the complete required clock count while CS remains
asserted.

Do not split the transaction into independent CS windows unless the
datasheet explicitly permits it.

------------------------------------------------------------------------

# 39. J5-to-J10 resistor problem

The EVAL-AD7616SDZ contains routing between the standalone J5 header and
the SDP J10 connector.

If the SDP board is not being used, unintended loading or contention
through the J10 path can interfere with debugging.

Analog Devices' standalone guide specifically recommends removing
appropriate 0 Ω resistors between J5 and J10 when necessary.

For a clean MCU-only setup:

``` text
EVAL-AD7616SDZ
      |
      +---- J5 ---- MCU
```

and not:

``` text
                 +---- MCU
                 |
AD7616 ---- J5 --+---- J10 ---- SDP
```

unless both interfaces are intentionally designed to coexist.

------------------------------------------------------------------------

# 40. Recommended first firmware milestone

Do this before FreeRTOS.

``` text
main()
 |
 +-- GPIO init
 |
 +-- SPI init
 |
 +-- AD7616 reset
 |
 +-- AD7616 interface check
 |
 +-- CONVST
 |
 +-- BUSY wait
 |
 +-- SPI read
 |
 +-- UART print
```

Output:

``` text
ADC A = 0xAAAA
ADC B = 0x5555
```

Once this works:

``` text
FreeRTOS
   |
   +-- ADC acquisition task
   |
   +-- SPI DMA
   |
   +-- ADC processing
   |
   +-- telemetry task
```

------------------------------------------------------------------------

# 41. Second milestone --- real analog signal

After interface-check mode is verified:

``` text
disable interface check
```

Then connect a known safe differential analog source.

For example, if configured for ±10 V:

``` text
0 V differential
```

should produce approximately:

``` text
0x0000
```

A positive input should produce a positive signed ADC code.

A negative input should produce a negative signed ADC code.

Never test an unknown external voltage source without verifying that it
is within the configured ADC input range and evaluation-board input
limits.

------------------------------------------------------------------------

# 42. Third milestone --- channel sequencing

Then implement:

``` text
V0A/V0B
V1A/V1B
V2A/V2B
...
V7A/V7B
```

Validate every channel independently.

Use a known reference signal or calibrated source.

Do not assume that because V0 works, all 16 channels are correctly
configured.

------------------------------------------------------------------------

# 43. Fourth milestone --- DMA

Move from:

``` text
HAL_SPI_TransmitReceive()
```

to:

``` text
HAL_SPI_TransmitReceive_DMA()
```

Then measure:

``` text
CPU utilization
DMA completion latency
sample-to-sample jitter
```

------------------------------------------------------------------------

# 44. Fifth milestone --- FreeRTOS

Recommended tasks:

``` text
AD7616_AcquisitionTask
AD7616_ProcessingTask
TelemetryTask
CommunicationTask
WatchdogTask
```

The acquisition task should have a higher priority than telemetry
formatting.

For example:

``` text
Acquisition       High
Processing        Medium-high
Communication     Medium
Telemetry         Medium
Diagnostics       Low
```

Exact priorities should be based on measured deadlines, not arbitrary
numbers.

------------------------------------------------------------------------

# 45. Throughput considerations

The AD7616 supports high throughput, but your total system rate is
determined by:

``` text
ADC conversion time
+
BUSY synchronization
+
SPI transfer time
+
DMA latency
+
processing time
+
communication time
```

For one-wire acquisition of two 16-bit values:

``` text
32 SCLK cycles / sample pair
```

At 10 MHz:

``` text
32 / 10 MHz
= 3.2 µs
```

of raw serial clock time.

At 1 MHz:

``` text
32 / 1 MHz
= 32 µs
```

The conversion itself and required timing must also be included.

Therefore, start with a low SPI frequency such as 1 MHz for bring-up,
then increase it after the waveform is proven.

------------------------------------------------------------------------

# 46. Why DMA matters

Without DMA:

``` text
CPU
 |
 +-- SPI transfer
 +-- wait
 +-- SPI transfer
 +-- wait
 +-- process
```

With DMA:

``` text
CPU
 |
 +-- start DMA
 |
 |       DMA
 |        |
 |        +---- SPI clock/data
 |
 +-- execute other RTOS work
 |
 +-- DMA completion interrupt
```

This is particularly important if your STM32U585 firmware is also
running:

``` text
motor control
load-cell processing
Bluetooth
UART
safety monitoring
watchdog
telemetry
```

------------------------------------------------------------------------

# 47. Recommended final data path

For your larger STM32U585 project:

``` text
              AD7616
                 |
                 | SPI + CONVST + BUSY
                 v
        +-------------------+
        | ADC Acquisition   |
        | FreeRTOS Task     |
        | SPI DMA           |
        +---------+---------+
                  |
                  v
        +-------------------+
        | ADC Processing    |
        |                   |
        | calibration       |
        | filtering         |
        | scaling           |
        +---------+---------+
                  |
                  v
        +-------------------+
        | Sample Buffer     |
        | / Queue           |
        +---------+---------+
                  |
                  v
        +-------------------+
        | Telemetry Task    |
        |                   |
        | timestamp         |
        | packet             |
        | sequence           |
        | CRC32              |
        +---------+---------+
                  |
          +-------+-------+
          |               |
          v               v
        UART            SPI/BLE
```

This architecture keeps the ADC timing deterministic and prevents
communications code from blocking acquisition.

------------------------------------------------------------------------

# 48. Production-level safeguards

The ADC driver should explicitly detect:

``` text
RESET timeout
BUSY timeout
SPI timeout
DMA error
invalid sample count
buffer overrun
configuration failure
unexpected interface-check result
```

Use an error enum:

``` c
typedef enum
{
    AD7616_OK = 0,
    AD7616_ERR_NULL,
    AD7616_ERR_RESET,
    AD7616_ERR_BUSY_TIMEOUT,
    AD7616_ERR_SPI,
    AD7616_ERR_DMA,
    AD7616_ERR_CONFIG,
    AD7616_ERR_INTERFACE_TEST
} AD7616_Status_t;
```

Do not silently return zero for failures.

`0x0000` is a valid ADC result, so:

``` c
return 0;
```

cannot be used as an error indicator for ADC data.

------------------------------------------------------------------------

# 49. Suggested driver API

A clean driver interface could be:

``` c
typedef struct
{
    SPI_HandleTypeDef *hspi;

    GPIO_TypeDef *cs_port;
    uint16_t cs_pin;

    GPIO_TypeDef *reset_port;
    uint16_t reset_pin;

    GPIO_TypeDef *convst_port;
    uint16_t convst_pin;

    GPIO_TypeDef *busy_port;
    uint16_t busy_pin;

} AD7616_Handle_t;
```

API:

``` c
AD7616_Status_t AD7616_Init(AD7616_Handle_t *dev);

AD7616_Status_t AD7616_Reset(AD7616_Handle_t *dev);

AD7616_Status_t AD7616_ReadRegister(
    AD7616_Handle_t *dev,
    uint8_t address,
    uint16_t *value);

AD7616_Status_t AD7616_WriteRegister(
    AD7616_Handle_t *dev,
    uint8_t address,
    uint16_t value);

AD7616_Status_t AD7616_StartConversion(
    AD7616_Handle_t *dev);

AD7616_Status_t AD7616_ReadConversion(
    AD7616_Handle_t *dev,
    uint16_t *channel_a,
    uint16_t *channel_b);

AD7616_Status_t AD7616_InterfaceTest(
    AD7616_Handle_t *dev);
```

This allows the application to remain independent of the ADC register
protocol.

------------------------------------------------------------------------

# 50. Important implementation rule

Do not start by implementing all AD7616 features.

Use this sequence:

``` text
Stage 1
-------
Power
RESET
CONVST
BUSY


Stage 2
-------
SER/PAR
SPI
SDOA
CS
SCLK


Stage 3
-------
Interface test
0xAAAA / 0x5555


Stage 4
-------
Real ADC
V0A / V0B


Stage 5
-------
Channel configuration


Stage 6
-------
Sequencer


Stage 7
-------
DMA


Stage 8
-------
FreeRTOS


Stage 9
-------
CRC


Stage 10
-------
High-speed continuous acquisition
```

This minimizes debugging dimensions.

------------------------------------------------------------------------

# 51. Practical wiring checklist

Before powering the system:

``` text
[ ] Common GND connected
[ ] AVDD checked
[ ] VDRIVE checked
[ ] RESET connected
[ ] CONVST connected
[ ] BUSY connected
[ ] SCLK connected
[ ] SDI connected
[ ] SDOA connected
[ ] CS connected
[ ] SER/PAR configured for serial
[ ] SER1W configured correctly
[ ] Software/hardware mode intentionally selected
[ ] J5/J10 routing checked
[ ] Analog input range configured
[ ] Analog source within range
```

------------------------------------------------------------------------

# 52. Recommended logic-analyzer test

Capture:

``` text
CH0 = CONVST
CH1 = BUSY
CH2 = CS
CH3 = SCLK
CH4 = MOSI
CH5 = MISO
```

Perform one conversion.

Save the capture.

The first target is:

``` text
CONVST -> BUSY -> CS -> SCLK -> MISO
```

Only after this waveform is correct should you increase the sample rate.

------------------------------------------------------------------------

# 53. Reference implementation strategy for STM32U585

For your existing STM32U585/FreeRTOS firmware, I would use:

``` text
SPI2
 |
 +-- AD7616

GPDMA
 |
 +-- SPI2_RX
 +-- SPI2_TX

GPIO
 |
 +-- AD7616_RESET
 +-- AD7616_CONVST
 +-- AD7616_CS
 +-- AD7616_BUSY

FreeRTOS
 |
 +-- AD7616Task
 +-- ADCProcessingTask
 +-- TelemetryTask
```

Keep the AD7616 driver independent of your prosthetic-knee telemetry
packet layer.

The resulting dependency should be:

``` text
AD7616 driver
     |
     v
ADC acquisition
     |
     v
ADC sample object
     |
     v
SmartLimb telemetry
```

not:

``` text
AD7616 driver
     |
     +---- SmartLimb packet
     +---- UART
     +---- BLE
     +---- display
```

The latter creates unnecessary coupling.

------------------------------------------------------------------------

# 54. Key references

## Analog Devices --- AD7616 product page

https://www.analog.com/en/products/ad7616.html

## AD7616 datasheet

https://www.analog.com/media/en/technical-documentation/data-sheets/ad7616.pdf

## EVAL-AD7616SDZ User Guide UG-1012

https://www.analog.com/media/en/technical-documentation/user-guides/EVAL-AD7616SDZ-7616-PSDZ-UG-1012.pdf

## Analog Devices standalone-mode guide

https://ez.analog.com/data_converters/precision_adcs/a/documents/DO13798/ad7616-evaluation-board-in-standalone-mode

## Analog Devices no-OS AD7616 driver

https://github.com/analogdevicesinc/no-OS/blob/main/drivers/adc/ad7616/ad7616.c

The no-OS driver is especially useful when implementing register access,
channel selection, range selection, reset, conversion reads, and CRC.

------------------------------------------------------------------------

# 55. Final recommended implementation

For your exact objective:

> **Acquire analog data from EVAL-AD7616SDZ and transfer the ADC data
> through SPI using STM32U585**

implement this first:

``` text
EVAL-AD7616SDZ
        |
        | V0A/V0B
        v
     AD7616
        |
        | CONVST
        | BUSY
        | CS
        | SCLK
        | SDI
        | SDOA
        v
    STM32U585
        |
        | SPI DMA
        v
   uint16_t adc[2]
        |
        v
 ADC processing task
        |
        v
 telemetry/second SPI/UART
```

The first successful data path should be:

``` text
V0A/V0B
   ↓
AD7616
   ↓
CONVST
   ↓
BUSY falling
   ↓
CS LOW
   ↓
32 SCLK clocks
   ↓
adc[0] = V0A
adc[1] = V0B
   ↓
CS HIGH
```

Then replace the blocking SPI transaction with DMA and integrate it into
FreeRTOS.

------------------------------------------------------------------------

## Critical takeaway

The hardest part of this project is **not writing
`HAL_SPI_TransmitReceive()`**.

The critical parts are:

1.  Correct EVAL-AD7616SDZ hardware modification.
2.  Correct `SER/PAR` and `SER1W` configuration.
3.  Correct software/hardware mode selection.
4.  Correct RESET timing.
5.  Correct `CONVST` → `BUSY` sequencing.
6.  Correct SPI clock timing.
7.  Keeping CS asserted for the required transaction.
8.  Correct 16-bit signed ADC interpretation.
9.  DMA synchronization.
10. Separating ADC acquisition from application communication.

Once the interface-check pattern `0xAAAA / 0x5555` is obtained reliably,
the remaining firmware development becomes substantially more
straightforward.
