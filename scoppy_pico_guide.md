# Scoppy-Pico Firmware: Complete Technical Documentation

## Overview

Scoppy-Pico is a C-based firmware designed for the Raspberry Pi Pico and RP2040 microcontroller that transforms the device into a fully functional digital oscilloscope. The project enables affordable oscilloscope functionality by combining the Pico's hardware capabilities with an Android application that displays and analyzes captured signals.

**Key Characteristics:**
- Target Hardware: Raspberry Pi Pico / RP2040 microcontroller
- Programming Language: C (96.2%) with CMake build system (3.8%)
- Communication: USB (native) or Wi-Fi (Pico W variant)
- Max Sample Rate: 500kS/s (stock), upgradable to 2.0MS/s
- Resolution: 12-bit ADC
- Input Voltage Range: 0-3.3V (native)
- Logic Analyzer Capability: 8-channel digital logic analyzer

---

## Architecture Overview

### System Components

```
┌─────────────────────────────────────────────────────┐
│         Android Device (Scoppy App)                 │
│    - Displays waveforms                             │
│    - Provides UI controls                           │
│    - Configures sampling parameters                 │
└─────────────────────┬───────────────────────────────┘
                      │ USB or Wi-Fi
                      │
┌─────────────────────┴───────────────────────────────┐
│      Raspberry Pi Pico / RP2040                      │
│  ┌──────────────────────────────────────────────┐  │
│  │  Scoppy-Pico Firmware (C)                    │  │
│  │  - ADC Control & Sampling                    │  │
│  │  - Data Acquisition                          │  │
│  │  - Communication Protocol                    │  │
│  │  - GPIO Management                           │  │
│  └──────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────┐  │
│  │  RP2040 Hardware                             │  │
│  │  - Dual-core ARM Cortex-M0+                  │  │
│  │  - 2x 12-bit ADCs (3 channels)               │  │
│  │  - 264KB SRAM                                │  │
│  │  - Clock: up to 133MHz                       │  │
│  │  - 26 GPIO pins                              │  │
│  └──────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘
                      │
                      ├─→ GPIO26 (ADC0, Channel 1)
                      ├─→ GPIO27 (ADC1, Channel 2)
                      └─→ GPIO6-13 (Logic Analyzer)
```

---

## Hardware Specifications

### RP2040 Processor Details

**Processor:**
- Dual-core ARM Cortex-M0+ running at 125MHz (default), up to 133MHz
- 264KB of internal SRAM
- 2MB of programmable flash memory

**Analog-to-Digital Converters (ADC):**
- Two independent 12-bit ADCs (technically 3 channels via MUX)
- Default clock source: 48MHz
- Conversion time: 96 clock cycles per sample
- Default maximum sample rate: 500,000 samples/second (48MHz ÷ 96)
- Can be overclocked for higher sample rates (up to 2.0MS/s)

**GPIO Pins Utilized:**
- GPIO26: ADC0 (Channel 1, main oscilloscope input)
- GPIO27: ADC1 (Channel 2, secondary oscilloscope input)
- GPIO29: Signal Generator Output (for testing/calibration)
- GPIO6-13: Logic Analyzer Inputs (8 digital channels)

**Communications:**
- USB 1.1 Full-Speed via native RP2040 USB controller
- Optional Wi-Fi via Pico W variant

---

## Firmware Operation

### Build System

The project uses CMake for cross-platform compilation:

```bash
git clone https://github.com/fhdm-dev/scoppy-pico.git
cd scoppy-pico
mkdir build
cd build
cmake ../pico
make
```

**Output:** A `.uf2` file that can be directly copied to the Pico's mass storage device

### Firmware Installation Process

1. **Enter Bootloader Mode:** Press and hold BOOTSEL button while connecting Pico to USB
2. **Mount as Storage:** Pico appears as "RPI-RP2" mass storage device
3. **Copy Firmware:** Drag the `.uf2` file onto the Pico's storage
4. **Auto-Reboot:** Device automatically reboots with Scoppy firmware
5. **Ready for Use:** Connect to Android device via USB-OTG or Wi-Fi

### Core Firmware Functions

#### 1. ADC Initialization & Control

**Purpose:** Configure and manage the RP2040's analog-to-digital converters

**Key Operations:**
- Initialize ADC hardware with appropriate clock source (48MHz default)
- Configure sampling mode (single-ended inputs 0-3.3V)
- Set up interrupts or DMA for data transfer
- Manage ADC clock multiplier (overclocking capability for higher sample rates)

**Sampling Modes:**
- **Standard Mode:** 500kS/s (one sample per 96 ADC clock cycles)
- **Overclocked Modes:** 1.3MS/s or 2.0MS/s (pushes ADC clock beyond datasheet limits)
- **Dual-Channel:** Sample rate halved when both channels enabled (250kS/s, 625kS/s, 1MS/s)

#### 2. Data Acquisition Engine

**Purpose:** Continuously sample signal data at configured rate

**Process:**
- ADC continuously converts analog input to 12-bit digital values
- Samples stored in circular buffer in SRAM (264KB available)
- Configurable trigger modes:
  - Free-running (continuous sampling)
  - Rising edge trigger
  - Falling edge trigger
  - Level triggering
- Pre-trigger buffering allows capture of signal before trigger event

**Buffer Management:**
- Circular buffer prevents memory overflow
- Newer samples overwrite oldest when memory full
- User can "freeze" capture via app command

#### 3. Communication Protocol

**Purpose:** Bi-directional USB/Wi-Fi communication with Android app

**Data Flow:**
- **Device → App:** Raw sample data, status information, capabilities
- **App → Device:** Configuration commands, trigger settings, sampling rate adjustment

**Protocol Features:**
- Real-time streaming of sample data
- Low-latency command response
- Status reporting (buffer state, trigger status)
- Configuration persistence across sessions

#### 4. Logic Analyzer Mode

**Purpose:** Digital logic signal capture (mode switch via app)

**GPIO Inputs:** GPIO 6-13 (8 digital channels)
- Each pin sampled as 3.3V logic level (high/low)
- Maximum sample rate: 25MS/s (standard) or 38MS/s (overclocked)
- Timing accuracy to microsecond level
- Synchronized trigger with oscilloscope mode

#### 5. Signal Generator

**Purpose:** Output test signals for oscilloscope calibration

**Pin:** GPIO22 (PWM output)

**Capabilities:**
- Square wave generation (up to 1.25MHz)
- Pulse-width modulated (PWM) 1kHz sine wave approximation
- Allows direct loopback testing (GPIO22 → GPIO26/27)
- Useful for AFE (Analog Front End) calibration

---

## Oscilloscope Operation Flow

### Data Acquisition Sequence

```
1. App requests sampling configuration
   ↓
2. Firmware configures ADC (sample rate, trigger mode)
   ↓
3. ADC begins continuous sampling into circular buffer
   ↓
4. Firmware waits for trigger condition (if enabled)
   ↓
5. Trigger fires → firmware marks sample position
   ↓
6. Firmware continues collecting post-trigger samples
   ↓
7. Buffer contains pre-trigger + post-trigger data
   ↓
8. App requests sample data
   ↓
9. Firmware streams sample data to Android device
   ↓
10. App displays waveform and measurements
```

### Measurement Capabilities

The Scoppy app provides real-time measurements of captured signals:

- **Voltage:** Peak, average, RMS across waveform
- **Frequency:** Calculated from period between triggers
- **Period/Time:** Duration of signal cycle or pulse
- **Duty Cycle:** Percentage of high time vs. total period
- **Rise/Fall Time:** Time for signal transition (for digital signals)

---

## ADC Sampling Deep Dive

### Standard Operation (500kS/s)

**Clock Calculation:**
```
ADC Clock: 48MHz (fixed per datasheet)
Cycles per sample: 96
Sample period: 96 / 48MHz = 2 microseconds
Sample rate: 1 / 2μs = 500,000 samples/second
```

**Nyquist Frequency:** 250kHz (can reliably capture signals up to this frequency)

**Practical Bandwidth:** ~600kHz when overclocked to 2.0MS/s

### Overclocking for Higher Sample Rates

**1.3MS/s Mode:**
- ADC clock increased to ~125MHz
- Maintains datasheet compliance margin
- Minimal accuracy loss

**2.0MS/s Mode:**
- ADC clock increased further (violates datasheet)
- Sample period: ~500 nanoseconds
- Achievable with negligible performance degradation
- Enables sampling of signals up to ~600kHz

### Two-Channel Operation

When both channels active:
- Samples alternate between channels
- Effective per-channel rate: half of configured rate
- Time-multiplexed measurement of two signals
- Synchronized timing between channels

---

## Input Signal Limitations

### Native Voltage Range
- **Minimum:** 0V (GND reference)
- **Maximum:** 3.3V (RP2040 limit)
- **Resolution:** 3.3V / 4096 levels = 0.805mV per step

### Addressing Limitations: Analog Front End (AFE)

For signals outside 0-3.3V range, external conditioning required:

**Simple Voltage Divider:**
- Measure 0-33V by dividing by 10
- Low cost, passive circuit

**High-Impedance AFE:**
- Buffers input signal to prevent loading
- Typical input impedance: 1MΩ
- Over/under voltage protection

**Multi-Range AFE:**
- Selectable gain/attenuation
- Typical ranges: ±330mV, ±3.3V, ±33V
- Software-selectable from app

---

## Firmware Code Structure

### Directory Organization

```
scoppy-pico/
├── pico/                 # Main firmware source
│   ├── CMakeLists.txt    # Build configuration
│   ├── src/              # C source files
│   │   ├── main.c        # Entry point
│   │   ├── adc.c         # ADC control
│   │   ├── usb.c         # USB communication
│   │   ├── capture.c     # Data acquisition
│   │   └── ...
│   └── include/          # Header files
└── scoppy/               # Shared code/documentation
```

### Key Modules

**1. main.c**
- Initialization sequence
- Hardware setup
- Event loop / interrupt handlers

**2. adc.c**
- ADC peripheral configuration
- Sampling rate control
- Overclocking parameters
- DMA setup for data transfer

**3. usb.c**
- USB device descriptor
- Protocol implementation
- Command parsing
- Data streaming

**4. capture.c**
- Circular buffer management
- Trigger detection
- Sample buffering logic
- Time-stamping

---

## Communication Protocol

### USB Connection

**Device Class:** Vendor-specific or CDC (Communications Device Class)

**Typical Commands:**
- `SET_SAMPLING_RATE`: Configure ADC clock and conversion parameters
- `START_CAPTURE`: Begin data acquisition
- `STOP_CAPTURE`: Halt sampling
- `GET_SAMPLES`: Request buffered sample data
- `SET_TRIGGER`: Configure trigger conditions
- `GET_STATUS`: Query device state

### Wi-Fi Connection (Pico W)

**Protocol:** Similar to USB but over TCP/UDP

**Advantages:**
- No wired connection needed
- Lower latency than USB in some cases
- Enables mobile-first workflow

---

## Performance Characteristics

### Timing Accuracy

- **Sample timing jitter:** <1% at standard rates
- **Trigger latency:** Single clock cycle (7.6ns at 131MHz)
- **Time-base accuracy:** Dependent on RP2040 clock accuracy

### Memory Usage

- **Circular Buffer:** Up to 264KB SRAM available
- **At 500kS/s:** ~266ms of waveform capture (500,000 × 2 bytes × 0.266)
- **At 2.0MS/s:** ~66ms of capture
- **Typical buffer:** 32-64KB, allowing 64-128ms capture

### Power Consumption

- **Active sampling:** ~50-100mA at 3.3V
- **Idle:** ~5mA
- **USB-powered:** Sufficient for single-channel operation
- **Note:** Dual-channel increases power consumption

---

## Key Features & Capabilities

### Oscilloscope Mode

- Real-time waveform capture and display
- Adjustable time-per-division (time scale)
- Adjustable volts-per-division (amplitude scale)
- Trigger modes: rising edge, falling edge, level
- Pre/post-trigger sample control
- Measurements: frequency, period, duty cycle, voltage stats

### Logic Analyzer Mode

- 8-channel digital signal capture
- Protocol analysis (SPI, I2C, UART, etc.)
- Timing resolution to microseconds
- State analysis and protocol decoding

### Signal Generator (Calibration)

- PWM-based square wave output
- Sine wave approximation via PWM
- Frequency range: DC to 1.25MHz
- Useful for testing and AFE calibration

---

## Typical Use Cases

1. **Audio Signal Analysis:** Measure and analyze audio frequency signals (up to ~20kHz)
2. **Digital Protocol Debugging:** Capture and decode SPI, I2C, UART communication
3. **Power Supply Ripple:** Monitor DC rail noise and transients
4. **Educational Experiments:** Learn oscilloscope operation and signal analysis
5. **Circuit Troubleshooting:** Quick verification of signal presence and timing
6. **Embedded Systems Testing:** Debug microcontroller output signals

---

## Limitations & Workarounds

| Limitation | Reason | Workaround |
|-----------|--------|-----------|
| 0-3.3V input only | RP2040 ADC limits | Build/buy AFE circuit |
| 500kS/s max (stock) | ADC clock limitation | Use overclocked mode (not guaranteed) |
| 12-bit resolution | RP2040 hardware limit | None (hardware limitation) |
| No negative voltage direct | ADC design | Use AC coupling + biasing in AFE |
| Input impedance load | ADC capacitance | Use high-impedance AFE |
| Limited SRAM | RP2040 spec | Trade off capture duration for fidelity |

---

## Development & Customization

### Required Tools

- **Compiler:** ARM GCC (arm-none-eabi-gcc)
- **Build System:** CMake 3.12+
- **SDK:** Raspberry Pi Pico SDK (pico-sdk)
- **Optional:** VS Code with C/C++ extensions

### Extending the Firmware

Common modifications:
- Increase overclocking limits for higher sample rates
- Add external ADC support for higher resolution
- Implement protocol-specific decoders
- Add SD card logging for long-term captures
- Custom triggering algorithms

---

## Conclusion

Scoppy-Pico transforms an inexpensive Raspberry Pi Pico into a capable digital oscilloscope by efficiently utilizing its hardware ADCs, GPIO, and high-speed USB interface. The firmware implements sophisticated signal acquisition, buffering, and communication while remaining compact enough to fit within the RP2040's flash memory. When paired with an Android device running the Scoppy app, it provides professional-grade oscilloscope functionality at a fraction of traditional equipment cost, making it ideal for hobbyists, students, and engineers.