\# STM32F407 Smart Controller



A modular embedded control system developed on the \*\*STM32F407VGT6\*\* using \*\*STM32CubeIDE\*\* and the \*\*STM32 HAL library\*\*.



The project integrates multiple MCU peripherals into a complete firmware application, including:



\- ADC sampling with DMA

\- UART command interface with DMA

\- I2C temperature sensing

\- SPI OLED display

\- PWM output control

\- External interrupt handling

\- System state machine

\- Fault detection and recovery



The firmware is implemented using a \*\*super-loop architecture without an RTOS\*\*, while DMA and interrupt mechanisms are used to reduce unnecessary CPU polling.



\---



\## Overview



The main goal of this project is to demonstrate practical embedded firmware development using common STM32 peripherals and software architecture techniques.



The controller supports two operating modes:



\- \*\*MANUAL mode\*\*  

&#x20; PWM duty cycle is controlled through UART commands.



\- \*\*AUTO mode\*\*  

&#x20; PWM duty cycle is automatically calculated according to the measured temperature.



The system also continuously monitors hardware status. A critical temperature-sensor fault forces the system into a safe `FAULT` state and disables the PWM output.



\---



\## Key Features



\### ADC + DMA



\- ADC1

\- 12-bit resolution

\- Continuous conversion

\- DMA-based acquisition

\- 32-sample averaging

\- Raw ADC value range: `0 \~ 4095`

\- Millivolt conversion available

\- DMA completion callback used to signal new data



\### UART + DMA Command Interface



\- USART2

\- 115200 baud

\- 8 data bits

\- 1 stop bit

\- No parity

\- No hardware flow control

\- DMA reception

\- UART IDLE detection

\- Human-readable ASCII commands

\- Command parsing and parameter validation



\### I2C Temperature Sensor



\- I2C1

\- 100 kHz

\- LM75 temperature sensor

\- Default 7-bit address: `0x48`

\- Temperature stored internally in units of `0.1 °C`

\- Sensor communication failure is treated as a critical fault



\### SPI OLED Display



\- SPI2

\- SSD1306-compatible 128 × 64 OLED

\- Displays:

&#x20; - Current operating mode

&#x20; - Temperature

&#x20; - ADC value

&#x20; - PWM duty cycle

&#x20; - Fault status



\### PWM Control



\- TIM2 Channel 2

\- PWM frequency: approximately `1 kHz`

\- Duty cycle range: `0 \~ 100%`

\- Manual UART control

\- Automatic temperature-based control



\### External Interrupt



\- USER button connected to `PA0`

\- Rising-edge EXTI

\- Interrupt callback only sets an event flag

\- Main loop performs the actual LED toggle operation



\### Fault Management



\- Centralized fault bitmask

\- Critical / non-critical fault distinction

\- Automatic transition to `FAULT` state for critical faults

\- PWM output forced to `0%` during critical fault

\- UART fault-clear command with hardware re-check



\---



\# System Architecture



```text

&#x20;                        +----------------------+

&#x20;                        |       User / PC      |

&#x20;                        |  Serial Terminal     |

&#x20;                        +----------+-----------+

&#x20;                                   |

&#x20;                                   | UART

&#x20;                                   v

&#x20;                        +----------------------+

&#x20;                        |   Protocol Handler   |

&#x20;                        | ASCII Command Parser |

&#x20;                        +----------+-----------+

&#x20;                                   |

&#x20;                                   v

+---------------------------------------------------------------+

|                       Application Layer                       |

|                                                               |

|   +---------------+   +---------------+   +----------------+ |

|   | State Machine |   | Fault Manager |   | Control Logic  | |

|   +---------------+   +---------------+   +----------------+ |

|                                                               |

+----------------------------+----------------------------------+

&#x20;                            |

&#x20;        +-------------------+-------------------+

&#x20;        |                   |                   |

&#x20;        v                   v                   v

+----------------+   +----------------+   +----------------+

| Analog Input   |   | Temperature    |   | PWM Control    |

| ADC + DMA      |   | LM75 / I2C     |   | TIM2 CH2       |

+----------------+   +----------------+   +----------------+

&#x20;        |                   |                                            |

&#x20;        |                                          |                                            |

&#x20;        +-------------------+--------------------+

&#x20;                            |

&#x20;                            v

&#x20;                   +----------------+

&#x20;                   | OLED Display   |

&#x20;                   | SSD1306 / SPI2 |

&#x20;                   +----------------+



&#x20;                            |

&#x20;                            v



&#x20;                    STM32 HAL Layer



&#x20;                            |

&#x20;                            v



&#x20;                     STM32F407VGT6

```



\---



\# Software Architecture



The firmware uses a modular structure.



Each major function is separated into its own `.c/.h` module.



```text

main.c

&#x20;|

&#x20;+-- AnalogInput

&#x20;|

&#x20;+-- TemperatureSensor

&#x20;|

&#x20;+-- PWM

&#x20;|

&#x20;+-- OLED

&#x20;|

&#x20;+-- Protocol

&#x20;|

&#x20;+-- SystemState

&#x20;|

&#x20;+-- FaultManager

```



This keeps peripheral-specific code separated from application control logic.



\---



\# Main Loop



The main application follows a super-loop architecture.



Conceptually:



```c

while (1)

{

&#x20;   AnalogInput\_Process();



&#x20;   App\_ProcessSensor();



&#x20;   App\_CheckCriticalFault();



&#x20;   App\_ProcessControl();



&#x20;   App\_ProcessDisplay();



&#x20;   App\_CheckCriticalFault();



&#x20;   ProcessButtonEvent();



&#x20;   ProcessUartCommand();

}

```



Interrupts and DMA callbacks mainly set flags, while the main loop performs the heavier application processing.



This reduces the amount of work performed inside interrupt service routines.



\---



\# Event-Driven Processing



Several peripherals use event flags.



\## ADC DMA



```text

ADC conversion

&#x20;     |

&#x20;     v

DMA fills 32-sample buffer

&#x20;     |

&#x20;     v

HAL\_ADC\_ConvCpltCallback()

&#x20;     |

&#x20;     v

adc\_dma\_ready = 1

&#x20;     |

&#x20;     v

AnalogInput\_Process()

&#x20;     |

&#x20;     v

Calculate 32-sample average

```



\---



\## UART DMA



```text

UART data received

&#x20;      |

&#x20;      v

DMA reception

&#x20;      |

&#x20;      v

UART IDLE event

&#x20;      |

&#x20;      v

HAL\_UARTEx\_RxEventCallback()

&#x20;      |

&#x20;      v

uart\_command\_ready = 1

&#x20;      |

&#x20;      v

Protocol\_ParseAscii()

&#x20;      |

&#x20;      v

App\_HandleCommand()

```



\---



\## External Interrupt



```text

USER button

&#x20;   |

&#x20;   v

PA0 EXTI

&#x20;   |

&#x20;   v

HAL\_GPIO\_EXTI\_Callback()

&#x20;   |

&#x20;   v

button\_event = 1

&#x20;   |

&#x20;   v

Main loop

&#x20;   |

&#x20;   v

Toggle LED

```



\---



\# System States



The controller contains five application states:



```text

INIT

IDLE

MANUAL

AUTO

FAULT

```



\## State Description



| State | Description |

|---|---|

| `INIT` | Firmware initialization |

| `IDLE` | System ready, PWM forced to 0% |

| `MANUAL` | PWM controlled by UART command |

| `AUTO` | PWM automatically controlled by temperature |

| `FAULT` | Critical fault state, PWM forced to 0% |



\---



\# State Machine



```text

&#x20;                    Power On

&#x20;                       |

&#x20;                       v

&#x20;                    +------+

&#x20;                    | INIT |

&#x20;                    +--+---+

&#x20;                       |

&#x20;         +-------------+-------------+

&#x20;         |                           |

&#x20;    No critical fault          Critical fault

&#x20;         |                           |

&#x20;         v                           v

&#x20;      +------+                    +-------+

&#x20;      | IDLE |<-------------------| FAULT |

&#x20;      +--+---+     CLEAR          +-------+

&#x20;         |

&#x20;         +-------------------+

&#x20;         |                   |

&#x20;         v                   v

&#x20;    +---------+          +------+

&#x20;    | MANUAL  |<-------->| AUTO |

&#x20;    +----+----+          +--+---+

&#x20;         |                  |

&#x20;         +--------+---------+

&#x20;                  |

&#x20;                 STOP

&#x20;                  |

&#x20;                  v

&#x20;               +------+

&#x20;               | IDLE |

&#x20;               +------+

```



Critical faults may force the controller into the `FAULT` state from any normal operating state.



\---



\# Operating Modes



\## IDLE Mode



In `IDLE` mode:



```text

PWM = 0%

```



The system continues monitoring sensors and processing UART commands.



\---



\## MANUAL Mode



In `MANUAL` mode, the user directly controls PWM duty through UART.



Example:



```text

MANUAL

PWM 50

```



Result:



```text

PWM duty = 50%

```



The `PWM` command is rejected if the system is not currently in `MANUAL` mode.



\---



\## AUTO Mode



In `AUTO` mode, temperature automatically determines PWM duty.



Control range:



```text

Temperature <= 25 °C

PWM = 0%



Temperature >= 40 °C

PWM = 100%

```



Between 25 °C and 40 °C, linear interpolation is used.



The equation is:



```text

&#x20;                 Temperature - 25

PWM (%) =  ------------------------------ × 100

&#x20;                      40 - 25

```



Example:



```text

25.0 °C  ->   0%

28.0 °C  ->  20%

32.5 °C  ->  50%

37.0 °C  ->  80%

40.0 °C  -> 100%

```



\---



\# UART Command Interface



\## Communication Settings



| Parameter | Value |

|---|---|

| Peripheral | USART2 |

| Baud rate | 115200 |

| Data bits | 8 |

| Stop bits | 1 |

| Parity | None |

| Flow control | None |

| Receive method | DMA + UART IDLE detection |



Commands are ASCII-based and human readable.



\---



\# Supported Commands



| Command | Description |

|---|---|

| `PING` | Test communication |

| `START` | Enter MANUAL mode |

| `STOP` | Stop PWM and return to IDLE |

| `MANUAL` | Enter MANUAL mode |

| `AUTO` | Enter AUTO mode |

| `PWM <0-100>` | Set PWM duty in MANUAL mode |

| `STATUS` | Read complete controller status |

| `TEMP` | Read temperature |

| `ADC` | Read ADC value |

| `CLEAR` | Clear faults and attempt recovery |

| `LED ON` | Turn external LED on |

| `LED OFF` | Turn external LED off |

| `LED TOGGLE` | Toggle external LED |



\---



\# UART Examples



\## Communication Test



Command:



```text

PING

```



Response:



```text

PONG

```



\---



\## Enter Manual Mode



Command:



```text

MANUAL

```



Response:



```text

OK MODE=MANUAL

```



\---



\## Set PWM Duty



Command:



```text

PWM 50

```



Response:



```text

OK PWM=50

```



If PWM is sent outside MANUAL mode:



```text

ERR MODE NOT MANUAL

```



\---



\## Start Command



Command:



```text

START

```



Response:



```text

OK MODE=MANUAL

```



`START` provides a simple way to enter MANUAL operation.



\---



\## Stop Controller



Command:



```text

STOP

```



Typical response:



```text

OK MODE=IDLE PWM=0

```



PWM is immediately set to zero.



\---



\## Automatic Mode



Command:



```text

AUTO

```



Response:



```text

OK MODE=AUTO

```



PWM is then automatically determined from LM75 temperature data.



\---



\## Read Temperature



Command:



```text

TEMP

```



Example response:



```text

TEMP=25.5C

```



If the temperature sensor has failed:



```text

ERR TEMP SENSOR

```



\---



\## Read ADC



Command:



```text

ADC

```



Example:



```text

ADC=2048

```



ADC resolution is 12-bit:



```text

0 \~ 4095

```



\---



\## Read Complete Status



Command:



```text

STATUS

```



Example:



```text

MODE=MANUAL TEMP=25.5C ADC=2048 PWM=50 FAULT=0x0000

```



The response contains:



```text

MODE

TEMP

ADC

PWM

FAULT

```



Example during a temperature sensor fault:



```text

MODE=FAULT TEMP=ERR ADC=2048 PWM=0 FAULT=0x0001

```



\---



\## LED Control



```text

LED ON

```



Response:



```text

OK LED=ON

```



\---



```text

LED OFF

```



Response:



```text

OK LED=OFF

```



\---



```text

LED TOGGLE

```



Response:



```text

OK LED=TOGGLE

```



The external LED is controlled through GPIO `PC14`.



\---



\# Fault Management



The firmware contains a centralized fault manager using a bitmask.



Currently used fault conditions include:



| Fault | Description | Critical |

|---|---|---|

| `FAULT\_TEMP\_SENSOR` | LM75 communication failure | Yes |

| `FAULT\_INVALID\_COMMAND` | Unsupported UART command | No |

| `FAULT\_INVALID\_PARAMETER` | Invalid UART parameter | No |



The temperature sensor fault is considered critical.



When a critical fault occurs:



```text

Critical fault detected

&#x20;       |

&#x20;       v

Set fault bit

&#x20;       |

&#x20;       v

Transition to FAULT state

&#x20;       |

&#x20;       v

PWM = 0%

```



This provides a basic fail-safe mechanism.



\---



\# Fault Recovery



Fault recovery is explicitly requested using:



```text

CLEAR

```



The firmware:



1\. Clears the current software fault flags.

2\. Re-checks the LM75 temperature sensor.

3\. Checks whether a critical fault still exists.

4\. Returns to `IDLE` only if the critical hardware check succeeds.

5\. Keeps PWM at `0%` during recovery.



Successful example:



```text

CLEAR

```



```text

OK FAULT=0x0000 MODE=IDLE

```



If the hardware fault still exists:



```text

ERR FAULT=0x0001

```



\---



\# ADC Acquisition



ADC1 operates in continuous conversion mode.



Configuration:



| Parameter | Value |

|---|---|

| ADC | ADC1 |

| Resolution | 12-bit |

| Channel | Channel 11 |

| Conversion | Continuous |

| DMA | Enabled |

| DMA samples | 32 |

| Sampling time | 56 cycles |

| ADC clock | PCLK2 / 4 |



The firmware averages 32 ADC samples:



```text

ADC sample 0  ┐

ADC sample 1  │

ADC sample 2  │

...           ├──> Average ──> ADC result

ADC sample 31 ┘

```



This provides a more stable ADC value than using only a single conversion.



The raw ADC range is:



```text

0 \~ 4095

```



The driver also provides a millivolt conversion based on a nominal 3.3 V reference:



```text

Voltage (mV) = ADC × 3300 / 4095

```



\---



\# PWM Configuration



PWM is generated using:



```text

TIM2 Channel 2

```



Configuration:



```text

Timer clock = 84 MHz

Prescaler   = 83

Period      = 999

```



Therefore:



```text

PWM frequency

= 84 MHz / (83 + 1) / (999 + 1)



= 1000 Hz

```



So the PWM output frequency is approximately:



```text

1 kHz

```



PWM duty range:



```text

0% \~ 100%

```



\---



\# Temperature Sampling



The LM75 is periodically read every:



```text

500 ms

```



The internal temperature representation is:



```text

Celsius × 10

```



Example:



```text

25.5 °C

```



is stored internally as:



```text

255

```



This avoids requiring floating-point arithmetic in the control logic.



\---



\# OLED Display



The OLED is updated every:



```text

200 ms

```



The display shows five main values:



```text

MODE: MANUAL

TEMP: 25.5C

ADC: 2048

PWM: 50%

FAULT: NONE

```



During a fault:



```text

MODE: FAULT

TEMP: ...

ADC: ...

PWM: 0%

FAULT: YES

```



The OLED is connected through SPI2.



Additional OLED control pins:



| Signal | STM32 Pin |

|---|---|

| CS | PE7 |

| DC | PE8 |

| RESET | PE9 |



\---



\# Hardware / Peripheral Summary



| Peripheral | Purpose |

|---|---|

| ADC1 | Analog input acquisition |

| DMA2 | ADC data transfer |

| USART2 | PC command interface |

| DMA1 | UART receive |

| I2C1 | LM75 temperature sensor |

| SPI2 | SSD1306 OLED |

| TIM2 CH2 | PWM generation |

| EXTI0 | USER button interrupt |

| PC14 GPIO | External LED |



The exact alternate-function pin assignments are stored in:



```text

STM32F407VGT6\_Smart\_Controller.ioc

```



\---



\# Important GPIO



| Pin | Function |

|---|---|

| PA0 | USER button / EXTI |

| PC14 | External LED |

| PE7 | OLED CS |

| PE8 | OLED DC |

| PE9 | OLED RESET |



Peripheral alternate-function pins for USART2, I2C1, SPI2, ADC1, and TIM2 are configured through STM32CubeMX and can be checked directly in the `.ioc` file.



\---



\# Clock Configuration



The MCU runs from the internal HSI oscillator.



```text

HSI = 16 MHz

```



PLL configuration:



```text

PLLM = 16

PLLN = 336

PLLP = 2

```



System clock:



```text

16 MHz / 16 × 336 / 2

= 168 MHz

```



Therefore:



```text

SYSCLK = 168 MHz

HCLK   = 168 MHz

PCLK1  = 42 MHz

PCLK2  = 84 MHz

```



\---



\# Project Structure



```text

STM32F407-Smart-Controller/

│

├── Core/

│   │

│   ├── Inc/

│   │   ├── analog\_input.h

│   │   ├── fault\_manager.h

│   │   ├── main.h

│   │   ├── oled.h

│   │   ├── protocol.h

│   │   ├── pwm.h

│   │   ├── stm32f4xx\_hal\_conf.h

│   │   ├── stm32f4xx\_it.h

│   │   ├── system\_state.h

│   │   └── temperature\_sensor.h

│   │

│   ├── Src/

│   │   ├── analog\_input.c

│   │   ├── fault\_manager.c

│   │   ├── main.c

│   │   ├── oled.c

│   │   ├── protocol.c

│   │   ├── pwm.c

│   │   ├── stm32f4xx\_hal\_msp.c

│   │   ├── stm32f4xx\_it.c

│   │   ├── syscalls.c

│   │   ├── sysmem.c

│   │   ├── system\_state.c

│   │   ├── system\_stm32f4xx.c

│   │   └── temperature\_sensor.c

│   │

│   └── Startup/

│       └── startup\_stm32f407vgtx.s

│

├── Drivers/

│   ├── CMSIS/

│   └── STM32F4xx\_HAL\_Driver/

│

├── STM32F407VGT6\_Smart\_Controller.ioc

├── STM32F407VGTX\_FLASH.ld

├── STM32F407VGTX\_RAM.ld

├── .gitignore

└── README.md

```



\---



\# Module Responsibilities



\## `analog\_input.c`



Responsible for:



\- Starting ADC DMA

\- Maintaining DMA sample buffer

\- Averaging 32 ADC samples

\- Providing raw ADC result

\- Converting ADC value to millivolts

\- Handling ADC conversion-complete callback



\---



\## `temperature\_sensor.c`



Responsible for:



\- LM75 initialization

\- I2C communication

\- Temperature register reading

\- Raw temperature decoding

\- Returning temperature in Celsius × 10



\---



\## `pwm.c`



Responsible for:



\- Starting TIM2 PWM

\- Setting duty cycle

\- Limiting duty to `0 \~ 100%`

\- Maintaining current duty value



\---



\## `protocol.c`



Responsible for:



\- UART ASCII command normalization

\- Command recognition

\- Parameter extraction

\- PWM value parsing

\- Invalid command detection

\- Invalid parameter detection



\---



\## `oled.c`



Responsible for:



\- SSD1306 initialization

\- SPI command/data transmission

\- Frame buffer management

\- Character rendering

\- String rendering

\- OLED screen update



\---



\## `system\_state.c`



Responsible for:



\- Application state storage

\- State transition validation

\- Current-state query



Application states:



```text

INIT

IDLE

MANUAL

AUTO

FAULT

```



\---



\## `fault\_manager.c`



Responsible for:



\- Setting fault flags

\- Clearing fault flags

\- Clearing all faults

\- Reading current fault bitmask

\- Detecting critical faults



\---



\## `main.c`



Responsible for high-level application coordination:



\- Peripheral initialization

\- Module initialization

\- Application scheduling

\- Sensor updates

\- UART command dispatch

\- State-based control

\- Fault handling

\- OLED updates

\- Button event processing



\---



\# Development Environment



| Item | Value |

|---|---|

| MCU | STM32F407VGT6 |

| Language | C |

| IDE | STM32CubeIDE |

| Configuration | STM32CubeMX |

| Firmware library | STM32 HAL |

| Architecture | Super-loop |

| RTOS | None |

| Debug interface | ST-LINK |

| Version control | Git |

| Repository hosting | GitHub |



\---



\# Hardware Requirements



Typical hardware used with this project:



\- STM32F407VGT6 development board

\- ST-LINK debugger/programmer

\- LM75 temperature sensor

\- SSD1306 SPI OLED display

\- Analog voltage source or potentiometer

\- External LED with suitable resistor

\- USB-to-UART adapter if required by the development board

\- Breadboard

\- Jumper wires

\- 3.3 V / 5 V power source as appropriate for each module



Always ensure that STM32 GPIO input voltage remains within the MCU's permitted voltage range.



\---



\# Build and Run



\## 1. Clone Repository



```bash

git clone https://github.com/YOUR\_USERNAME/STM32F407-Smart-Controller.git

```



Replace:



```text

YOUR\_USERNAME

```



with your GitHub username.



\---



\## 2. Open STM32CubeIDE



Import the project using:



```text

File

→ Import

→ General

→ Existing Projects into Workspace

```



Select the cloned project folder.



\---



\## 3. Check CubeMX Configuration



The hardware configuration is stored in:



```text

STM32F407VGT6\_Smart\_Controller.ioc

```



Open the `.ioc` file in STM32CubeIDE to inspect peripheral and pin configuration.



\---



\## 4. Build



Build the project using:



```text

Project

→ Build Project

```



or the STM32CubeIDE build button.



\---



\## 5. Flash Firmware



Connect the STM32 board through ST-LINK and use:



```text

Run

```



or:



```text

Debug

```



to program the firmware.



\---



\# Serial Terminal Setup



A serial terminal such as:



\- PuTTY

\- SSCOM

\- Tera Term

\- RealTerm



can be used.



UART settings:



```text

Baud rate : 115200

Data bits : 8

Stop bits : 1

Parity    : None

Flow Ctrl : None

```



Send one complete ASCII command at a time.



Example:



```text

PING

```



Expected response:



```text

PONG

```



\---



\# Suggested Demo Sequence



A simple project demonstration can follow this sequence.



\## 1. Communication



```text

PING

```



Expected:



```text

PONG

```



\---



\## 2. Check Initial State



```text

STATUS

```



Expected system state:



```text

MODE=IDLE

```



\---



\## 3. Manual PWM Control



```text

MANUAL

PWM 25

PWM 50

PWM 75

PWM 100

```



Verify PWM duty using:



\- Oscilloscope

\- Logic analyzer

\- LED brightness

\- Suitable driver/load



\---



\## 4. ADC DMA



Change the analog input voltage and repeatedly send:



```text

ADC

```



Confirm that ADC value changes between:



```text

0 \~ 4095

```



\---



\## 5. Temperature Sensor



Send:



```text

TEMP

```



Example:



```text

TEMP=26.0C

```



The OLED should show the same temperature information.



\---



\## 6. AUTO Mode



Send:



```text

AUTO

```



Change the LM75 temperature.



Verify that PWM automatically changes between:



```text

0% \~ 100%

```



according to the 25 °C to 40 °C control range.



\---



\## 7. Fault Handling



Disconnect or interrupt communication with the LM75 sensor.



The controller should:



```text

Set FAULT\_TEMP\_SENSOR

&#x20;       |

&#x20;       v

Enter FAULT state

&#x20;       |

&#x20;       v

Force PWM to 0%

```



Check:



```text

STATUS

```



\---



\## 8. Fault Recovery



Reconnect the sensor and send:



```text

CLEAR

```



If sensor communication is restored:



```text

OK FAULT=0x0000 MODE=IDLE

```



\---



\# Firmware Concepts Demonstrated



This project demonstrates practical use of:



\- STM32 peripheral configuration

\- GPIO

\- EXTI

\- ADC

\- DMA

\- UART

\- UART IDLE detection

\- I2C

\- SPI

\- Hardware timers

\- PWM

\- Interrupt callbacks

\- Event flags

\- Periodic scheduling using `HAL\_GetTick()`

\- Modular C firmware design

\- State-machine architecture

\- Fault bitmasks

\- Fail-safe output handling

\- ASCII protocol parsing

\- Parameter validation

\- Hardware abstraction using STM32 HAL

\- Git version control



\---



\# Design Decisions



\## DMA for ADC



ADC data is collected into a memory buffer using DMA rather than reading every conversion directly from the CPU.



Benefits:



\- Reduced CPU involvement

\- Continuous data acquisition

\- Easy multi-sample averaging



\---



\## DMA + IDLE for UART RX



UART reception uses DMA with IDLE detection rather than continuously polling the UART peripheral.



Benefits:



\- Reduced CPU polling

\- Variable-length command reception

\- Cleaner separation between communication and application logic



\---



\## Lightweight Interrupt Callbacks



Interrupt callbacks mainly set event flags.



Example:



```text

Interrupt

&#x20;  |

&#x20;  v

Set flag

&#x20;  |

&#x20;  v

Return quickly

&#x20;  |

&#x20;  v

Main loop handles event

```



This avoids placing large amounts of application logic inside interrupt handlers.



\---



\## Integer Temperature Representation



Temperature uses:



```text

Celsius × 10

```



instead of floating-point values.



Example:



```text

25.5 °C = 255

```



Benefits:



\- Simple arithmetic

\- Predictable embedded behavior

\- No floating-point requirement in control calculations



\---



\## Centralized Fault Manager



Fault conditions are stored in one bitmask instead of using unrelated error variables across modules.



This makes it easier to:



\- Add new faults

\- Check system health

\- Report errors over UART

\- Define critical faults

\- Implement safe-state behavior



\---



\# Current Status



The first functional firmware version is complete.



Implemented and tested functions include:



\- ADC DMA acquisition

\- ADC averaging

\- UART DMA command reception

\- ASCII command parsing

\- LM75 temperature acquisition

\- SSD1306 OLED output

\- Manual PWM control

\- Automatic PWM control

\- GPIO LED control

\- External button interrupt

\- State-machine control

\- Fault detection

\- Critical-fault safe state

\- Fault recovery command

\- UART status reporting



\---



\# Possible Future Improvements



Potential future extensions include:



\- FreeRTOS version

\- UART TX using DMA

\- Command-line help command

\- More robust UART stream framing

\- Sensor fault debounce / retry mechanism

\- Watchdog timer

\- Non-volatile configuration storage

\- Additional sensor support

\- Motor/fan driver integration

\- Unit testing for protocol and state-machine modules

\- More detailed diagnostic fault codes



These features are intentionally left as future extensions rather than being required for the current functional version.



\---



\# Project Purpose



This project was developed as a practical embedded firmware exercise and portfolio project.



The focus is not only on making individual peripherals work, but also on integrating them into a structured firmware system with:



```text

Peripheral Drivers

&#x20;       +

DMA / Interrupts

&#x20;       +

Application State Machine

&#x20;       +

Control Logic

&#x20;       +

Fault Management

&#x20;       +

Human-Readable Communication Interface

```



The project is intended to demonstrate skills relevant to junior embedded / firmware engineering positions.



\---



\# Notes



STM32 HAL and CMSIS source files included under the `Drivers/` directory remain subject to their respective STMicroelectronics license terms.



The application-specific firmware is located mainly under:



```text

Core/Inc/

Core/Src/

```

