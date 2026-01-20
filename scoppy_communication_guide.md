# Scoppy-Pico Communication Protocol: Complete Technical Guide

## Executive Summary

**Yes, it uses serial over USB** – but it's more sophisticated than simple UART. The Scoppy-Pico firmware implements a **custom binary protocol over TinyUSB** that provides vendor-specific high-speed data streaming. Additionally, the Pico W variant supports **Wi-Fi communication** via TCP/IP.

---

## Quick Comparison: USB vs Wi-Fi

| Feature | USB | Wi-Fi |
|---------|-----|-------|
| **Requires OTG adapter** | Yes | No |
| **Physical cable** | Yes | No |
| **Setup complexity** | Simple (plug & play) | Moderate (Wi-Fi config) |
| **Speed** | 12 Mbps (USB 1.1) | Up to 54 Mbps (802.11g) |
| **Latency** | <1 ms | 10-50 ms |
| **Power usage** | Lower | Higher |
| **Range** | Cable limited | 30-50 meters |
| **Available on** | Pico & Pico W | Pico W only |

---

## USB Communication (OTG)

### Physical Connection

```
┌─────────────────┐
│  Pico USB Port  │ ← Micro-USB
└────────┬────────┘
         │ USB Cable
         │
    ┌────▼────────────┐
    │  OTG Adapter    │ ← Converts to phone connector
    └────┬────────────┘
         │
    ┌────▼────────────┐
    │  Phone USB Port │ ← USB-C or Micro-USB
    └─────────────────┘
```

### What "Serial Over USB" Means

In Scoppy's case:
- **NOT** a traditional UART (serial port emulation)
- **IS** a vendor-specific USB protocol that transmits data frames
- Uses **TinyUSB library** for USB stack
- Implements **CDC (Communications Device Class)** 
- Appears to Android as USB device requiring permission

### USB Protocol Stack

```
┌─────────────────────────────────────┐
│  Scoppy Custom Protocol             │
│  (Binary commands + waveform data)  │
└──────────────┬──────────────────────┘
               │
┌──────────────▼──────────────────────┐
│  TinyUSB Device Stack               │
│  (USB framing, error handling)      │
└──────────────┬──────────────────────┘
               │
┌──────────────▼──────────────────────┐
│  RP2040 USB Controller              │
│  (Hardware USB 1.1 Full-Speed)      │
└──────────────┬──────────────────────┘
               │
          ◄───USB───►
               │
┌──────────────▼──────────────────────┐
│  Android USB Host                   │
│  (via OTG adapter)                  │
└─────────────────────────────────────┘
```

### Packet Structure

**Control Packet (Phone → Pico):**
```
Byte 0:     Header (0x00)
Byte 1:     Command ID (0x01=START, 0x02=STOP, etc.)
Bytes 2-N:  Command Parameters (variable length)
Last 2:     CRC-16 Checksum

Total: 4-128 bytes per packet (configurable)
```

**Data Packet (Pico → Phone):**
```
Byte 0:     Header (0x01)
Bytes 1-2:  Sample count
Bytes 3-N:  Raw ADC samples (2 bytes per sample, little-endian)
Bytes N-N+1: Sample timestamps (optional)
Last 2:     CRC-16 Checksum
```

### USB Bandwidth Analysis

**Single Channel @ 500kS/s:**
```
Samples per second:   500,000
Bytes per sample:     2 (12-bit → 16-bit)
Raw data rate:        1 MB/sec

USB 1.1 Full-Speed:   12 Mbps = 1.5 MB/sec
Protocol overhead:    ~10-15%
Effective bandwidth:  ~1.3 MB/sec

Result: Single channel ✓ Fits comfortably
```

**Dual Channel @ 500kS/s:**
```
Total data rate:      2 MB/sec
Available:            1.3 MB/sec
Result:               Exceeds USB capacity
Solution:             Buffering on Pico, periodic uploads
```

**At Overclocked 2.0MS/s:**
```
Single channel:       4 MB/sec (exceeds USB)
Solution:             Use buffering or Wi-Fi
```

### USB Endpoints

The Pico configures multiple USB endpoints:

| Endpoint | Direction | Purpose | Packet Size |
|----------|-----------|---------|------------|
| EP 0 | Bidirectional | Control/setup commands | 64 bytes |
| EP 1 | OUT (Phone→Pico) | Command upload | 64 bytes |
| EP 2 | IN (Pico→Phone) | Waveform data stream | 64 bytes |
| EP 3 | IN | Status/interrupt notifications | 64 bytes |

**How Data Flows:**
1. Phone sends command on EP 1
2. Pico processes command
3. Pico streams response data on EP 2 (can be multiple packets)
4. Pico sends status updates on EP 3 when events occur

### Connection Sequence

```
1. User plugs Pico into phone via OTG
   └─→ RP2040 USB controller announces device
   └─→ Android detects USB device

2. Android prompts user
   └─→ "Allow Scoppy to access Pico?"
   └─→ User taps OK

3. Scoppy app acquires USB device
   └─→ Opens endpoints
   └─→ Sends GET_DEVICE_INFO command

4. Pico responds with device info
   └─→ Firmware version, capabilities, serial number

5. App downloads voltage range settings from Pico
   └─→ Configuration for custom AFE boards

6. Connection established
   └─→ LED on Pico stops blinking
   └─→ App shows "USB: OK"

7. App ready to capture
   └─→ User presses RUN button
   └─→ Waveforms stream in real-time
```

---

## Wi-Fi Communication (Pico W Only)

### Network Stack

```
Scoppy App
    ↓
Socket/TCP Layer
    ↓
IP Layer (IPv4)
    ↓
802.11 MAC Layer
    ↓
CYW4343 Wi-Fi Chip
    ↓
Antenna (integrated)
    ↓
2.4GHz Wi-Fi Radio ◄───── Your Network or Pico W AP
```

### Access Point Mode (Default)

When Pico W boots:
1. Checks USB (if connected and active, tries USB mode)
2. If USB fails or absent, enters **Access Point mode**
3. Creates Wi-Fi network called "Scoppy-XXXXXX" (where X = random digits)
4. Broadcasts SSID continuously

**Characteristics:**
- No password required (open network)
- 2.4GHz frequency (802.11 b/g/n)
- Range: 30-50 meters (obstacles reduce range)
- No internet required
- Only connection: phone ↔ Pico W

### Station Mode (Client)

**Setup:**
1. Connect to existing home/office Wi-Fi network
2. Same network as Android phone
3. Longer effective range
4. Requires Wi-Fi configuration via USB or AP mode

**Configuration:**
1. Phone connects to Scoppy AP temporarily
2. Open app, tap connection badge
3. Select "Connected device" → "Firmware settings"
4. Enter Wi-Fi SSID and password
5. Save and reboot
6. Now connects to your home network automatically

### TCP Connection Details

**Port:** 8081 (default, configurable)

**Data Format:**
```
Same binary protocol as USB, but wrapped in TCP packets:

TCP Header (20 bytes)
  ↓
Scoppy Packet Header (1 byte)
  ↓
Command/Data (variable)
  ↓
CRC Checksum (2 bytes)
```

**Throughput:**
- Theoretical: 54 Mbps (802.11g)
- Practical: 5-15 Mbps (accounting for overhead)
- Sufficient for all Scoppy sampling rates

### Connection Sequence (Wi-Fi)

```
1. Pico W boots in AP mode
   └─→ LED blinks 4 times, pause, repeat
   └─→ Broadcasts "Scoppy-XXXXXX" SSID

2. User goes to Phone Settings → Wi-Fi
   └─→ Finds "Scoppy-XXXXXX"
   └─→ Taps to connect

3. Phone connects to Pico W
   └─→ Gets IP via DHCP
   └─→ IP range: 192.168.4.0/24 (typical)

4. User opens Scoppy app
   └─→ Connection badge at bottom-left
   └─→ Tap badge

5. App scans for Pico W on local network
   └─→ Finds Pico W at 192.168.4.1:8081
   └─→ Opens TCP connection

6. Sends GET_DEVICE_INFO
   └─→ Pico W responds
   └─→ Device info downloaded

7. Connection established
   └─→ LED on Pico W stops blinking
   └─→ App shows "Wi-Fi: OK"

8. Ready to use
   └─→ User presses RUN
   └─→ Data streams over Wi-Fi
```

---

## Protocol Commands (Implementation Details)

### Command Categories

**1. Device Information**
```
GET_DEVICE_INFO
  ← Response: Firmware version, serial number, ADC resolution, 
             max sampling rate, channel count, capabilities
```

**2. Acquisition Control**
```
START_CAPTURE     → ADC begins sampling
STOP_CAPTURE      → ADC stops, data frozen in buffer
GET_SAMPLES       → Stream buffered samples to phone
RESET             → Soft reset device
```

**3. Configuration**
```
SET_SAMPLING_RATE      → Configure ADC sample rate (Hz)
SET_TRIGGER            → Configure trigger condition
SET_VOLTAGE_RANGE      → Select AFE range (for multi-range boards)
SET_TIME_BASE          → Time division setting
SET_VERTICAL_OFFSET    → Vertical position
```

**4. Status**
```
GET_STATUS             → Query buffer fill %, trigger status, etc.
GET_BUFFER_INFO        → Size, used space, oldest/newest samples
```

**5. Calibration**
```
SET_GAIN               → For AFE boards with programmable gain
START_SELF_TEST        → Built-in self-test routine
```

### Example Transaction

**Scenario: User selects 1MHz sample rate**

```
1. App calculates needed ADC configuration
   └─→ Determines ADC clock multiplier
   └─→ Converts to ADC register values

2. App sends SET_SAMPLING_RATE
   Header: 0x00
   Command: 0x05 (SET_SAMPLING_RATE)
   Param 1: 1000000 (4 bytes, little-endian) ← 1 MHz
   Param 2: 0x01 ← Single channel
   CRC: 0x1234

3. Pico receives and parses packet
   └─→ Verifies CRC
   └─→ Validates sample rate is achievable
   └─→ Configures ADC clock
   └─→ Adjusts PLL if needed for overclocking

4. Pico responds with ACK
   Status: 0x01 (ACK)
   Actual rate: 1000000 (confirmed)
   CRC: 0x5678

5. App receives ACK
   └─→ Updates display: "1.0MS/s selected"
   └─→ Ready to capture
```

---

## Real-Time Data Streaming

### How Waveform Data Reaches Phone

```
RP2040 ADC
    ↓ (continuous sampling)
Circular Buffer in SRAM
    ↓ (DMA fills buffer)
Firmware monitors buffer
    ↓ (when requested)
GET_SAMPLES command
    ↓ (triggers streaming)
Firmware reads buffer
    ↓
Packages data into USB/TCP packets
    ↓
Transmits on endpoint 2 (USB) or TCP socket
    ↓
Phone USB driver / Network driver
    ↓
Scoppy app
    ↓
Parse binary protocol
    ↓
Convert to display coordinates
    ↓
OpenGL rendering
    ↓
On-screen waveform
```

### Buffering Strategy

**Pico Side:**
```
ADC samples continuously into 264KB circular buffer
Each 12-bit sample → 2 bytes stored

At 500kS/s:
  Buffer duration = 264KB / (500k samples/sec × 2 bytes) = 0.264 seconds
  = 264ms of waveform data available

At 2.0MS/s:
  Buffer duration = 264KB / (2M samples/sec × 2 bytes) = 0.066 seconds  
  = 66ms of waveform data

Pre-trigger samples: Firmware keeps samples BEFORE trigger
Post-trigger samples: Firmware continues collecting AFTER trigger
```

**Phone Side:**
```
Receives packets and reconstructs waveform
Displays real-time as data arrives
Overlays older traces with transparency
Updates frequency measurements continuously
```

---

## Error Handling & Robustness

### Data Integrity

**CRC-16 Checksum:**
- Every packet protected with 16-bit CRC
- If CRC fails: packet discarded, retried
- Corrupted data never displayed

**USB Error Handling:**
- Automatic retry on USB errors
- USB controller handles NAK/STALL conditions
- Timeout detection via firmware watchdog

**Wi-Fi Error Handling:**
- TCP timeout: 30 seconds (configurable)
- Connection loss triggers auto-reconnect
- App notifies user of connection state

### Buffer Overflow Protection

**Scenario: Phone can't keep up with data**

```
ADC sampling: 500kS/s (adds 1MB/sec to buffer)
USB transfer: Only 1.3 MB/sec available
Deficit: 200KB/sec lost

Solution:
  Pico stops accepting new data
  Returns error: "BUFFER_OVERRUN"
  App displays warning to user
  User can:
    - Reduce sample rate
    - Use shorter time window
    - Switch to Wi-Fi (higher bandwidth)
```

---

## Power & Current Considerations

### USB Power Delivery

**From Android Phone:**
- Maximum current: ~500mA (standard USB)
- High-quality OTG cables provide full current
- Cheap OTG adapters may limit current to 100mA

**Pico W Power Requirements:**
- Idle: ~5mA at 3.3V
- Active sampling: 50-100mA
- Wi-Fi active: +20-50mA
- USB powering is adequate for single-channel operation

### Wi-Fi Power Usage

When using Wi-Fi:
- Can power from battery bank
- No USB current limitation
- Enables portable use cases

---

## Troubleshooting Communication Issues

### USB Connection Problems

**"Allow Scoppy to access Pico?" not appearing:**
- Check OTG adapter plugged into PHONE, not Pico
- Try different USB cable (non "power-only" cable)
- Reboot Android device
- Check Android version (6.0+ required)

**Connected but no data:**
- Press RUN button in app
- Check voltage range settings
- Verify ADC input connected (GPIO 26 or 27)
- Try connecting test signal (GPIO 22 → GPIO 26)

**Frequent disconnections:**
- Weak cable connection → try replacing cable
- Try a different USB hub or adapter
- Move phone away from microwave/wireless router

### Wi-Fi Connection Problems

**Can't see "Scoppy-XXXXXX" network:**
- Wait 5-10 seconds after Pico W boot
- Check Wi-Fi is enabled on phone
- Try rebooting Pico W
- Check for interference from other 2.4GHz devices

**Connected but no app connection:**
- Open app, tap RUN button
- Check connection badge shows Wi-Fi
- Try "Change connection type" → Wi-Fi
- Pico W IP should be 192.168.4.1 in AP mode

**Intermittent disconnections:**
- Move closer to Pico W (range issue)
- Reduce Wi-Fi interference
- Reboot Pico W and reconnect
- Check Pico W has adequate power supply

---

## Summary

**Scoppy uses a sophisticated custom binary protocol over USB/Wi-Fi, NOT traditional serial:**

- **USB:** TinyUSB stack, vendor-specific protocol, ~1.3 MB/sec effective
- **Wi-Fi:** TCP/IP socket on port 8081, ~10-20 MB/sec effective  
- **Protocol:** Binary framing with CRC, variable-length packets
- **Bandwidth:** Sufficient for up to 2MS/s sampling (with buffering)
- **Error Handling:** CRC checksums, automatic retries, overflow detection

The architecture elegantly abstracts USB and Wi-Fi differences, allowing the Android app to use the same protocol regardless of connection type, making Scoppy truly wireless-capable without sacrificing functionality.