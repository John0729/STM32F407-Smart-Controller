<div align="center">

# STM32F407 Smart Controller

### Multi-Peripheral Embedded Control System

**STM32F407VGT6 · C · STM32 HAL · DMA · UART · I2C · SPI · PWM · EXTI**

A modular embedded firmware project integrating sensor acquisition,  
command control, automatic PWM regulation, OLED monitoring,  
state management, and fault handling.

</div>

---

## Overview

This project implements a complete embedded control system on the
**STM32F407VGT6**.

Instead of demonstrating peripherals independently, the project integrates
multiple STM32 peripherals into a single application with a structured
firmware architecture.

### Highlights

- ADC continuous sampling with **DMA**
- UART command interface using **DMA + IDLE detection**
- LM75 temperature sensing through **I2C**
- SSD1306 OLED display through **SPI**
- Hardware PWM generation with **TIM2**
- USER button input using **EXTI**
- `INIT / IDLE / MANUAL / AUTO / FAULT` state machine
- Centralized fault manager
- Automatic fail-safe PWM shutdown
- Human-readable ASCII command interface
- Modular `.c / .h` firmware structure

---

## System Architecture

```mermaid
flowchart TD

    PC[PC / Serial Terminal]
    UART[UART Protocol Handler]
    APP[Application Control]
    STATE[State Machine]
    FAULT[Fault Manager]

    ADC[ADC + DMA]
    TEMP[LM75 / I2C]
    OLED[SSD1306 / SPI]
    PWM[TIM2 PWM]
    BUTTON[PA0 EXTI]

    PC --> UART
    UART --> APP

    BUTTON --> APP

    APP --> STATE
    APP --> FAULT

    ADC --> APP
    TEMP --> APP

    APP --> PWM
    APP --> OLED
```

---

## Operating Modes

| Mode | Description |
|---|---|
| `INIT` | System initialization |
| `IDLE` | System ready, PWM output disabled |
| `MANUAL` | PWM controlled through UART |
| `AUTO` | PWM automatically controlled by temperature |
| `FAULT` | Critical fault state, PWM forced to 0% |

### State Flow

```mermaid
stateDiagram-v2
    [*] --> INIT

    INIT --> IDLE: Initialization OK
    INIT --> FAULT: Critical fault

    IDLE --> MANUAL
    IDLE --> AUTO

    MANUAL --> IDLE: STOP
    AUTO --> IDLE: STOP

    MANUAL --> AUTO
    AUTO --> MANUAL

    IDLE --> FAULT: Critical fault
    MANUAL --> FAULT: Critical fault
    AUTO --> FAULT: Critical fault

    FAULT --> IDLE: CLEAR + Recovery OK
```

---

## Automatic PWM Control

In `AUTO` mode, PWM duty cycle is calculated according to the LM75
temperature measurement.

| Temperature | PWM |
|---|---:|
| ≤ 25 °C | 0% |
| 32.5 °C | 50% |
| ≥ 40 °C | 100% |

For temperatures between 25 °C and 40 °C:

```text
PWM (%) = (Temperature - 25) × 100 / 15
```

This creates a simple linear temperature-control profile.

---

## UART Command Interface

USART2 provides a human-readable command interface.

### Configuration

```text
Baud Rate : 115200
Data Bits : 8
Stop Bits : 1
Parity    : None
Flow Ctrl : None
RX Method : DMA + UART IDLE
```

### Commands

| Command | Description |
|---|---|
| `PING` | Communication test |
| `START` | Enter MANUAL mode |
| `STOP` | Return to IDLE and stop PWM |
| `MANUAL` | Enter MANUAL mode |
| `AUTO` | Enter AUTO mode |
| `PWM <0-100>` | Set PWM duty |
| `STATUS` | Read system status |
| `TEMP` | Read temperature |
| `ADC` | Read ADC value |
| `CLEAR` | Clear fault and attempt recovery |
| `LED ON` | Turn LED on |
| `LED OFF` | Turn LED off |
| `LED TOGGLE` | Toggle LED |

### Example

```text
> PING
PONG

> MANUAL
OK MODE=MANUAL

> PWM 50
OK PWM=50

> STATUS
MODE=MANUAL TEMP=25.5C ADC=2048 PWM=50 FAULT=0x0000
```

---

## Peripheral Integration

| Peripheral | Configuration | Function |
|---|---|---|
| ADC1 | 12-bit + DMA | Analog signal acquisition |
| DMA2 | Circular transfer | ADC sample buffering |
| USART2 | 115200 + DMA | Command interface |
| I2C1 | 100 kHz | LM75 temperature sensor |
| SPI2 | Master | SSD1306 OLED |
| TIM2 CH2 | ~1 kHz | PWM generation |
| EXTI0 | Rising edge | USER button |
| GPIO PC14 | Output | External LED |

---

## ADC + DMA

ADC1 continuously samples an analog input.

A DMA buffer stores:

```text
32 samples
```

After the DMA transfer completes, the firmware calculates the average:

```mermaid
flowchart LR

    ADC[ADC1]
    DMA[DMA Buffer<br/>32 Samples]
    AVG[Average]
    VALUE[ADC Result]

    ADC --> DMA
    DMA --> AVG
    AVG --> VALUE
```

Raw ADC range:

```text
0 ~ 4095
```

The driver also provides voltage conversion:

```text
Voltage (mV) = ADC × 3300 / 4095
```

---

## UART DMA Reception

UART reception uses DMA together with UART IDLE detection.

```mermaid
flowchart LR

    RX[UART RX]
    DMA[DMA Buffer]
    IDLE[UART IDLE Event]
    FLAG[Command Ready]
    PARSER[ASCII Parser]
    APP[Application]

    RX --> DMA
    DMA --> IDLE
    IDLE --> FLAG
    FLAG --> PARSER
    PARSER --> APP
```

This avoids continuously polling the UART peripheral.

---

## Fault Management

The firmware uses a centralized fault bitmask.

Current fault conditions include:

| Fault | Critical |
|---|---|
| Temperature sensor communication failure | Yes |
| Invalid UART command | No |
| Invalid parameter | No |

A critical temperature sensor failure triggers:

```mermaid
flowchart LR

    ERROR[Sensor Failure]
    FAULT[Set Fault]
    STATE[Enter FAULT]
    PWM[PWM = 0%]

    ERROR --> FAULT
    FAULT --> STATE
    STATE --> PWM
```

This provides a basic fail-safe mechanism.

### Recovery

The user can issue:

```text
CLEAR
```

The firmware clears the fault flags and re-checks the temperature sensor.

If the sensor responds correctly:

```text
FAULT → IDLE
```

Otherwise the controller remains in the fault condition.

---

## OLED Monitoring

The SSD1306 OLED displays system information in real time.

Example:

```text
MODE: MANUAL
TEMP: 25.5C
ADC : 2048
PWM : 50%
FAULT: NONE
```

SPI2 is used for OLED communication.

Control pins:

| Signal | Pin |
|---|---|
| CS | PE7 |
| DC | PE8 |
| RESET | PE9 |

---

## Hardware Overview

### MCU

**STM32F407VGT6**

Clock configuration:

```text
SYSCLK : 168 MHz
HCLK   : 168 MHz
PCLK1  : 42 MHz
PCLK2  : 84 MHz
```

### Main Components

- STM32F407VGT6 development board
- LM75 temperature sensor
- SSD1306 SPI OLED
- Analog input / potentiometer
- External LED
- ST-LINK debugger
- USB-UART adapter
- Breadboard and jumper wires

---

## Software Structure

```text
Core/
├── Inc/
│   ├── analog_input.h
│   ├── fault_manager.h
│   ├── oled.h
│   ├── protocol.h
│   ├── pwm.h
│   ├── system_state.h
│   └── temperature_sensor.h
│
└── Src/
    ├── analog_input.c
    ├── fault_manager.c
    ├── main.c
    ├── oled.c
    ├── protocol.c
    ├── pwm.c
    ├── system_state.c
    └── temperature_sensor.c
```

### Module Responsibilities

| Module | Responsibility |
|---|---|
| `analog_input` | ADC DMA acquisition and averaging |
| `temperature_sensor` | LM75 I2C communication |
| `pwm` | PWM generation and duty control |
| `oled` | SSD1306 display driver |
| `protocol` | UART ASCII command parser |
| `system_state` | State transition management |
| `fault_manager` | Fault detection and storage |
| `main` | Application scheduling and coordination |

---

## Firmware Architecture

The project currently uses a **super-loop architecture without an RTOS**.

```mermaid
flowchart TD

    LOOP[Main Loop]

    ADC[Process ADC]
    SENSOR[Read Temperature]
    FAULT1[Check Fault]
    CONTROL[Process Control]
    DISPLAY[Update OLED]
    UART[Process UART]
    BUTTON[Process Button]

    LOOP --> ADC
    ADC --> SENSOR
    SENSOR --> FAULT1
    FAULT1 --> CONTROL
    CONTROL --> DISPLAY
    DISPLAY --> UART
    UART --> BUTTON
    BUTTON --> LOOP
```

DMA and interrupts handle asynchronous peripheral events, while application
logic remains primarily in the main loop.

---

## Design Principles

### Keep Interrupts Lightweight

Interrupt callbacks mainly set flags:

```c
button_event = 1;
```

The main loop performs the actual application processing.

This keeps interrupt execution short and predictable.

### Separate Drivers from Application Logic

Peripheral-specific logic is separated into dedicated modules instead of
placing everything inside `main.c`.

### Fail-Safe Output

Critical faults force:

```text
PWM = 0%
```

to prevent the actuator from continuing to operate when sensor feedback is
unavailable.

### Integer-Based Temperature

Temperature is stored internally as:

```text
Celsius × 10
```

Example:

```text
25.5 °C → 255
```

This allows control calculations without requiring floating-point arithmetic.

---

## Build

### Requirements

- STM32CubeIDE
- STM32 HAL
- ST-LINK

Clone the repository:

```bash
git clone https://github.com/YOUR_USERNAME/STM32F407-Smart-Controller.git
```

Import into STM32CubeIDE:

```text
File
→ Import
→ General
→ Existing Projects into Workspace
```

Then:

```text
Build Project
→ Connect ST-LINK
→ Run / Debug
```

The CubeMX hardware configuration is stored in:

```text
STM32F407VGT6_Smart_Controller.ioc
```

---

## Demo

### Suggested Test Flow

```text
1. PING
2. STATUS
3. MANUAL
4. PWM 25
5. PWM 50
6. PWM 100
7. ADC
8. TEMP
9. AUTO
10. Disconnect LM75
11. STATUS
12. Reconnect LM75
13. CLEAR
```

### Demo Video

> Demo video will be added here.

### Hardware Setup

> Hardware photo and wiring diagram will be added here.

---

## Skills Demonstrated

This project demonstrates practical experience with:

`STM32` · `Embedded C` · `HAL` · `GPIO` · `EXTI` · `ADC` · `DMA`  
`UART` · `I2C` · `SPI` · `PWM` · `Timers` · `State Machine`  
`Fault Handling` · `Serial Protocol` · `Git`

---

## Development Environment

| | |
|---|---|
| MCU | STM32F407VGT6 |
| IDE | STM32CubeIDE |
| Language | C |
| Framework | STM32 HAL |
| Architecture | Super-loop |
| RTOS | None |
| Version Control | Git / GitHub |

---

<div align="center">

### STM32F407 Smart Controller

**Embedded firmware portfolio project**

</div>