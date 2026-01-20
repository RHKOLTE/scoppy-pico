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
- GPIO 0 & 1: UART TX & RX (diagnostic output, error reporting)
- GPIO 2 & 3: CH1 Voltage Range Selection (binary encoding)
- GPIO 4 & 5: CH2 Voltage Range Selection (binary encoding)
- GPIO 6-13: Logic Analyzer Inputs (8 digital channels)
- GPIO 14: Wi-Fi Status LED output (Pico W only)
- GPIO 15: Trigger Status LED output
- GPIO 22: Test Signal Output (PWM 1kHz, 50% duty)
- GPIO 26: ADC0 (Channel 1, main oscilloscope input)
- GPIO 27: ADC1 (Channel 2, secondary oscilloscope input)

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

## GPIO 2, 3, 4, 5: Voltage Range Selection (AFE Control)

### Purpose

Yes, exactly as you suspected! GPIO 2, 3, 4, and 5 are used to select voltage ranges and control the Analog Front End (AFE) gain/attenuation. These pins allow the firmware to communicate with external amplifier/attenuator circuitry to measure signals beyond the native 0-3.3V range.

### Configuration

**Channel 1 Range Selection:**
- GPIO 2: Voltage Range Bit 1 (CH1)
- GPIO 3: Voltage Range Bit 0 (CH1)

**Channel 2 Range Selection:**
- GPIO 4: Voltage Range Bit 1 (CH2)
- GPIO 5: Voltage Range Bit 0 (CH2)

### Binary Encoding System

The two GPIO pins per channel form a 2-bit binary number that selects between multiple voltage ranges:

**Channel 1 (GPIO 2 & 3):**
```
GPIO 2 (Bit 1) | GPIO 3 (Bit 0) | Range ID | Example Range
    LOW        |     LOW        |    0     | 0-10V
    LOW        |     HIGH       |    1     | 0-5V
    HIGH       |     LOW        |    2     | 0-2V
    HIGH       |     HIGH       |    3     | 0-1V
```

**Channel 2 (GPIO 4 & 5):** (Same pattern as CH1)
```
GPIO 4 (Bit 1) | GPIO 5 (Bit 0) | Range ID | Example Range
    LOW        |     LOW        |    0     | 0-10V
    LOW        |     HIGH       |    1     | 0-5V
    HIGH       |     LOW        |    2     | 0-2V
    HIGH       |     HIGH       |    3     | 0-1V
```

### How It Works

**Two Operation Modes:**

#### Mode 1: Input (Manual Range Selection)
- GPIO 2/3/4/5 configured as **inputs with pull-down resistors**
- External mechanical switch or circuit sets logic levels
- Firmware reads the pins to determine currently selected range
- Used when hardware manually selects range

#### Mode 2: Output (Automatic Range Selection)
- GPIO 2/3/4/5 configured as **outputs**
- Firmware/app controls the logic levels based on user's Volts/Div setting
- External analog switch (like CD4052) responds to these pins
- Used for automatic range switching as user adjusts sensitivity

### Practical Example

**Simple Multi-Range AFE Design:**

```
Input Signal (0-30V)
        │
        ├─→ [Attenuator 1] ──→ 0-30V to 0-3.3V (Divide by ~10)
        │        (Divider)
        │
        ├─→ [Attenuator 2] ──→ 0-10V to 0-3.3V (Divide by ~3)
        │        (Divider)
        │
        ├─→ [Attenuator 3] ──→ 0-3V to 0-3.3V (Direct/Buffer)
        │        (Buffer)
        │
        └─→ [Analog Mux] ────→ GPIO 26 (ADC)
             (CD4052 or similar)
             Control: GPIO 2, 3
```

**How it selects ranges:**

- GPIO 2=LOW, GPIO 3=LOW → Mux selects Attenuator 1 (0-30V range)
- GPIO 2=LOW, GPIO 3=HIGH → Mux selects Attenuator 2 (0-10V range)
- GPIO 2=HIGH, GPIO 3=LOW → Mux selects Attenuator 3 (0-3V range)
- etc.

### Software Control Flow

**When user changes Volts/Div in app:**

```
1. User adjusts Volts/Div slider in Scoppy app
   ↓
2. App calculates optimal voltage range needed
   ↓
3. App sends command to firmware with new range ID (0-3)
   ↓
4. Firmware decodes range ID into binary:
   - Range 0: Set GPIO2=0, GPIO3=0
   - Range 1: Set GPIO2=0, GPIO3=1
   - Range 2: Set GPIO2=1, GPIO3=0
   - Range 3: Set GPIO2=1, GPIO3=1
   ↓
5. Firmware drives GPIO 2 & 3 to new logic levels
   ↓
6. External analog mux/switch responds to GPIO levels
   ↓
7. Different attenuator selected, ADC input voltage scaled appropriately
   ↓
8. App now displays waveform with full scale resolution
```

### Optional Third Bit (GPIO 7 and 11)

For **up to 8 voltage ranges** (instead of 4), an optional third bit can be enabled:

- **GPIO 7:** CH1 Voltage Range Bit 2 (optional)
- **GPIO 11:** CH2 Voltage Range Bit 2 (optional)

With 3 bits, range IDs 0-7 are possible, allowing 8 different voltage ranges per channel.

```
GPIO 2 | GPIO 3 | GPIO 7 | Range ID
 LOW   |  LOW   |  LOW   |    0     → 0-100V
 LOW   |  HIGH  |  LOW   |    1     → 0-30V
 HIGH  |  LOW   |  LOW   |    2     → 0-10V
 HIGH  |  HIGH  |  LOW   |    3     → 0-3V
 LOW   |  LOW   |  HIGH  |    4     → 0-1V
 LOW   |  HIGH  |  HIGH  |    5     → 0-300mV
 HIGH  |  LOW   |  HIGH  |    6     → 0-100mV
 HIGH  |  HIGH  |  HIGH  |    7     → 0-30mV
```

### Voltage Range Configuration in Scoppy App

Each voltage range is configured with:

- **Min Voltage:** Input voltage that results in 0V at ADC
- **Max Voltage:** Input voltage that results in 3.3V at ADC
- **Vref 1X (V):** Optional reference voltage for 1X probe setting
- **Vref 10X (V):** Optional reference voltage for 10X probe setting

**Example Configuration:**
```
Range 0: Min=-10V, Max=+10V (after AFE conditioning)
Range 1: Min=-5V, Max=+5V
Range 2: Min=-1V, Max=+1V
Range 3: Min=-0.3V, Max=+0.3V
```

The firmware stores these settings and uses them to convert raw ADC readings back to actual input voltages displayed in the app.

### Real-World AFE Examples

The Scoppy documentation provides several AFE designs:

1. **Simple Voltage Divider** - Single range, passive, cheapest
2. **High-Impedance AFE** - Buffers input, prevents loading effects
3. **Multi-Range Attenuator** - Selectable ranges via switches
4. **Auto-Ranging AFE** - Uses GPIO 2-5 pins for automatic range selection
5. **FSCOPE-500K Board** - Commercial option with integrated AFE

### Default Behavior (No AFE)

If GPIO 2/3/4/5 are not connected:
- Pins read as LOW (due to pull-down resistors)
- Range ID = 0 is always selected
- Only default 0-3.3V range is available
- This is typical for basic Scoppy setups

### Design Considerations for AFE Builders

When designing an AFE using these pins:

- **Pull-down resistors (10-100k)** on each pin ensure safe default state
- **Current limiting resistors** protect from accidental short circuits
- **Analog switches** (CD4052, CD4066, etc.) respond to GPIO logic levels
- **Op-amp buffers** maintain high input impedance
- **Precision resistors** ensure accurate voltage scaling
- **Over-voltage protection** (diodes, Zeners) prevents damage
- **Proper grounding** essential for accurate measurements

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

---

## Undocumented & Potentially Available GPIO Pins

### Pins NOT Used by Scoppy Firmware

The Scoppy firmware uses only a subset of the Raspberry Pi Pico's available GPIO pins. Many pins are **completely unused** and available for expansion or additional purposes.

**Unused GPIO Pins:**
```
GPIO 6 - GPIO 13:  Used by Logic Analyzer (inputs)
GPIO 16, 17, 18, 19, 20, 21:  ✓ COMPLETELY AVAILABLE
GPIO 24, 25, 28, 29:  ✓ AVAILABLE (with caveats - see below)
```

### Completely Available Pins (Safe to Use)

These GPIO pins are **not used by any Scoppy functionality** and can be repurposed for custom applications:

| GPIO | Physical Pin | Available? | Safe to Use? | Notes |
|------|-------------|-----------|------------|-------|
| 16 | 21 | ✓ YES | ✓ YES | General purpose I/O, PWM capable |
| 17 | 22 | ✓ YES | ✓ YES | General purpose I/O, PWM capable |
| 18 | 24 | ✓ YES | ✓ YES | General purpose I/O, PWM capable |
| 19 | 25 | ✓ YES | ✓ YES | General purpose I/O, PWM capable |
| 20 | 26 | ✓ YES | ✓ YES | General purpose I/O, PWM capable |
| 21 | 27 | ✓ YES | ✓ YES | General purpose I/O, PWM capable |

**Use Cases for Available Pins:**
- Custom LED indicators
- Button inputs for user control
- Serial communication (SPI, I2C, UART)
- PWM outputs for motor control
- Additional sensor inputs
- Custom hardware expansion

### Pins Available with Constraints

**GPIO 24 (Physical Pin 31):**
- Function: VBUS sense pin (detects USB power)
- Available: ✓ YES (but not typical)
- Use Case: Can be read to detect if USB is connected
- Safe: Generally avoid modifying

**GPIO 25 (Physical Pin 32):**
- Function: On-board LED control (standard Pico)
- Available: ✓ YES
- Status: May blink during operation
- Safe: Can be used but note LED will blink

**GPIO 28 (Physical Pin 34, ADC input):**
- Function: ADC2 input (analog pin)
- Available: ✓ YES
- Safe: ✓ YES for analog measurements
- Note: Can read 0-3.3V analog signals if enabled

**GPIO 29 (Physical Pin 35, ADC input):**
- Function: ADC3 input + VSYS voltage sense
- Available: ✓ PARTIAL
- Safe: Can measure VSYS voltage (system input)
- Note: Used for internal voltage monitoring

### Reserved Internal Pins (NOT Exposed)

These GPIO pins are **not accessible** on the Pico header pins:

| GPIO | Function | Accessible? |
|------|----------|------------|
| GPIO 23 | SMPS Power Save Control | ✗ NO |
| GPIO 24 | VBUS Sense | ✗ NO (internal only) |
| GPIO 25 | On-board LED | ✓ YES (on standard Pico) |
| GPIO 29 | VSYS / ADC3 | ✓ YES (ADC, not GPIO header) |

### Pico W Specific Notes

**Wi-Fi Module Pins:**
The Pico W includes an Infineon CYW4343 Wi-Fi/Bluetooth chip that uses some pins internally:
- Not accessible for general use
- These pins are controlled by the Wi-Fi firmware
- Do NOT attempt to use these pins for other purposes

**LED Control (Pico W):**
- GPIO 25 (LED) still available
- Additionally, GPIO 14 used for Wi-Fi status indication

### Complete GPIO Status Table

| GPIO | Function | Used by Scoppy | Available? | Notes |
|------|----------|---------------|-----------|-------|
| 0 | UART TX | YES | NO | Debug UART output |
| 1 | UART RX | YES | NO | Debug UART input |
| 2 | CH1 Range Bit 1 | YES | NO | Voltage range selection |
| 3 | CH1 Range Bit 0 | YES | NO | Voltage range selection |
| 4 | CH2 Range Bit 1 | YES | NO | Voltage range selection |
| 5 | CH2 Range Bit 0 | YES | NO | Voltage range selection |
| 6-13 | Logic Analyzer | YES | NO | Digital signal inputs (8 channels) |
| 14 | Wi-Fi Status LED | YES (Pico W) | NO | Wi-Fi connection indicator |
| 15 | Trigger LED | YES | NO | Trigger event indicator |
| 16 | — | NO | ✓ YES | Fully available |
| 17 | — | NO | ✓ YES | Fully available |
| 18 | — | NO | ✓ YES | Fully available |
| 19 | — | NO | ✓ YES | Fully available |
| 20 | — | NO | ✓ YES | Fully available |
| 21 | — | NO | ✓ YES | Fully available |
| 22 | Test Signal Output | YES | NO | 1kHz PWM square wave |
| 23 | SMPS Power Save | Internal | NO | Not accessible |
| 24 | VBUS Sense | Internal | NO | Not accessible |
| 25 | On-board LED | Available | ~ | Use with caution |
| 26 | ADC0 (CH1 input) | YES | NO | Oscilloscope channel 1 |
| 27 | ADC1 (CH2 input) | YES | NO | Oscilloscope channel 2 |
| 28 | ADC2 | Available | ✓ YES | Can be used as analog input |
| 29 | ADC3/VSYS sense | Available | ~ | Internal voltage monitoring |

### Potential Firmware Modifications

If customizing the Scoppy firmware, these GPIO configurations could be modified:

**GPIO 22 (Test Signal):**
- Could be redirected to different pin
- Could be repurposed if test signal not needed
- Currently PWM on slice 3, channel B

**GPIO 14/15 (Status LEDs):**
- Optional functionality
- Can be disabled if pins needed for other purposes
- Could be moved to available pins (16-21)

**Logic Analyzer Pins (6-13):**
- Fixed by design but could be remapped in firmware
- Would require recompilation
- Not recommended for standard users

### Safety Considerations

When using available GPIO pins:

1. **Voltage Levels:** All GPIO are 3.3V only
   - Do NOT apply 5V or higher
   - Use logic level shifters for non-3.3V circuits

2. **Current Limits:**
   - Per pin: Maximum 12mA
   - Total all pins: Maximum 50mA combined
   - Use current limiting resistors for LEDs

3. **Protection:**
   - Include series resistors (100-1k ohms) on inputs
   - Protect against over-voltage with resistor dividers
   - Use diode clamps for sensitive circuits

4. **Pull-up/Pull-down:**
   - Configure appropriate pull-resistors in firmware
   - Unused pins should be set to input with pull-down

### Practical Example: Custom Expansion

**Using GPIO 16-21 for Additional Functionality:**

```
GPIO 16 → External LED (with 220Ω resistor)
GPIO 17 → Push Button (with 10k pull-up)
GPIO 18 → I2C SCL (for external sensor)
GPIO 19 → I2C SDA (for external sensor)
GPIO 20 → SPI Clock (for SD card module)
GPIO 21 → SPI MOSI (for SD card module)
```

This allows adding data logging, external controls, or sensor interfaces while Scoppy operates normally.

---

## Conclusion

Scoppy-Pico transforms an inexpensive Raspberry Pi Pico into a capable digital oscilloscope by efficiently utilizing its hardware ADCs, GPIO, and high-speed USB interface. The firmware implements sophisticated signal acquisition, buffering, and communication while remaining compact enough to fit within the RP2040's flash memory. When paired with an Android device running the Scoppy app, it provides professional-grade oscilloscope functionality at a fraction of traditional equipment cost, making it ideal for hobbyists, students, and engineers.

The firmware uses only **15 GPIO pins out of 26 available**, leaving **GPIO 16-21** (and optionally 28) completely free for custom expansions, user projects, or future enhancements without conflicting with core Scoppy functionality.