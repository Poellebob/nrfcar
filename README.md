# NRF Car Communication Protocol

This document provides a detailed explanation of the wireless communication protocol between the `nrfcontroller` (remote) and the `nrfbot` (car) using nRF24L01 radio modules.

## System Overview

The system implements a **unidirectional wireless control link**:

- **Controller (TX)** Continuously broadcasts control commands
- **Car (RX)** Listens and responds to received commands

### Communication Architecture

```
Controller (nrfcontroller)          Car (nrfbot)
┌──────────────────┐              ┌──────────────────┐
│  radioNumber: 0  │  ───write─→  │  radioNumber: 1  │
│  Writes to:      │   "1Node"    │  Reads from:     │
│    "1Node"       │              │    "1Node"       │
│                  │              │                  │
│  Reads from:     │  ←───────    │  Writes to:      │
│    "2Node"       │   (unused)   │    "2Node"       │
└──────────────────┘              └──────────────────┘
```

**Key Concept:** Both devices define the same two addresses (`"1Node"` and `"2Node"`), but they use **opposite** `radioNumber` values:
- Controller uses `radioNumber = 0` → writes to `address[0]` = `"1Node"`
- Car uses `radioNumber = 1` → reads from `address[!1]` = `address[0]` = `"1Node"`

This creates a matched TX/RX pair on the same channel.

---

## Controller (`nrfcontroller/src/main.cpp`)

The controller reads joystick and button inputs, packages them into a payload, and transmits them wirelessly.

### Hardware Configuration

```cpp
#define CE_PIN 9        // Chip Enable pin for nRF24
#define CSN_PIN 10      // SPI Chip Select pin

// Analog inputs
#define x_axis A0       // Joystick X (throttle control)
#define y_axis A1       // Joystick Y (steering control)

// Digital button inputs
#define up_button 2
#define down_button 4
#define left_button 5
#define right_button 3
#define start_button 6
#define select_button 7
#define analog_button 8
```

### Radio Setup

```cpp
uint8_t address[][6] = {"1Node", "2Node"};
bool radioNumber = 0;   // Use first address for writing
bool role = true;       // true = Transmitter mode

// In setup():
radio.openWritingPipe(address[radioNumber]);      // Write to "1Node"
radio.openReadingPipe(1, address[!radioNumber]);  // Listen on "2Node" (unused)

if (role)
  radio.stopListening();  // Transmit mode - don't listen
```

**Explanation:**
- `openWritingPipe(address[0])` sets up transmission to `"1Node"`
- `openReadingPipe(1, address[1])` sets up listening on `"2Node"` (not used in current implementation, but available for future bidirectional communication)
- `stopListening()` puts the radio in TX mode, continuously ready to transmit

### Payload Structure

The controller sends a **9-element integer array** every 10ms:

```cpp
int16_t payload[9];

payload[0] = analogRead(x_axis);     // 󰮠 Throttle (0-1023)
payload[1] = analogRead(y_axis);     // 󰓁 Steering (0-1023)
payload[2] = digitalRead(up_button);     // 󰁝 Button states (0 or 1)
payload[3] = digitalRead(down_button);   // 󰁅
payload[4] = digitalRead(left_button);   // 󰁍
payload[5] = digitalRead(right_button);  // 󰁔
payload[6] = digitalRead(start_button);  // 󰐊
payload[7] = digitalRead(select_button); // 󰒓
payload[8] = digitalRead(analog_button); // 󰳽
```

### Transmission Loop

```cpp
void loop() {
  // Read all inputs
  payload[0] = analogRead(x_axis);
  payload[1] = analogRead(y_axis);
  payload[2] = digitalRead(up_button);
  // ... (remaining buttons)
  
  // Transmit entire payload
  radio.write(&payload, sizeof(payload));
  
  delay(10);  // 100Hz update rate
}
```

**How it works:**
1. **Read all inputs** from joysticks and buttons
2. **Pack into array** - analog values are 10-bit (0-1023), buttons are binary (0/1)
3. **Transmit** - `radio.write()` sends all 18 bytes (9 × 2-byte integers) in one packet
4. **Wait 10ms** - creates a 100Hz control loop

---

## Car (`nrfbot/src/main.cpp`)

The car receives control packets and uses them to drive its motors.

### Hardware Configuration

```cpp
// Motor control pins
#define MTR0f 5   // Motor 0 forward
#define MTR0b 4   // Motor 0 backward
#define MTR0e 2   // Motor 0 enable (PWM speed)
#define MTR1f 7   // Motor 1 forward
#define MTR1b 6   // Motor 1 backward
#define MTR1e 3   // Motor 1 enable (PWM speed)

// Sensor pins (defined but not used in current code)
#define SL 13     // Sensor Left
#define SR 12     // Sensor Right
#define SML 11    // Sensor Middle Left
#define SM 10     // Sensor Middle
#define SMR 9     // Sensor Middle Right

Car self = Car(MTR0f, MTR1f, MTR0b, MTR1b, MTR0e, MTR1e);
```

**Note:** The `Car` class (from `motor.h`) handles the low-level motor control logic.

### Radio Setup

```cpp
uint8_t address[][6] = {"1Node", "2Node"};
bool radioNumber = 1;   // Use second address for writing
bool role = false;      // false = Receiver mode
int16_t payload[9];     // Buffer to receive data

// In setup():
radio.openWritingPipe(address[radioNumber]);      // Write to "2Node" (unused)
radio.openReadingPipe(1, address[!radioNumber]);  // Listen on "1Node" ✓

if (role)
  radio.stopListening();
else
  radio.startListening();  // Receive mode - actively listening
```

**Critical Logic:**
- Car uses `radioNumber = 1`, so `address[radioNumber]` = `"2Node"`
- But it **reads** from `address[!radioNumber]` = `address[0]` = `"1Node"`
- This matches where the controller is **writing** to!

### Reception and Motor Control Loop

```cpp
void loop() {
  if (radio.available()) {  // Check if data received
    // Read the payload
    radio.read(&payload, sizeof(payload));
    
    // Debug output
    Serial.print(payload[0]);   // Print throttle
    Serial.println(payload[1]); // Print steering
    
    // Drive the car using first two payload values
    self.drive(payload[0], payload[1]);
  }
  delay(10);  // Check every 10ms
}
```

**How it works:**
1. **Check for data** - `radio.available()` returns true if a packet is ready
2. **Read payload** - `radio.read()` copies received data into the `payload` array
3. **Extract controls** - Only uses `payload[0]` (throttle) and `payload[1]` (steering)
4. **Drive motors** - `self.drive(throttle, steering)` translates joystick values into motor speeds

### Motor Control Logic

The `drive()` function (implemented in `motor.h`) interprets the joystick values:

- **Throttle (`payload[0]`)** - Controls forward/backward movement
  - Values near 512 (center): stop
  - Values > 512: forward motion
  - Values < 512: backward motion
  
- **Steering (`payload[1]`)** - Controls differential speed between motors
  - Values near 512 (center): straight
  - Values > 512: turn right (slow down right motor)
  - Values < 512: turn left (slow down left motor)

**Example:** If throttle=700 and steering=600:
- Both motors move forward (throttle > 512)
- Right motor slightly slower than left (steering > 512)
- Result: Car moves forward while turning right

---

## Data Flow Summary

```
 10ms Cycle:
 ┌────────────────────────────────────────────────────────────┐
 │                                                            │
 │  Controller:                                               │
 │  1. Read joysticks → [512, 487]                            │
 │  2. Read buttons   → [0,0,0,0,1,0,0]                       │
 │  3. Pack payload   → [512, 487, 0,0,0,0,1,0,0]             │
 │  4. Transmit       → 󰤃 RF signal                          │
 │                                                            │
 │  ─────────── 2.4GHz wireless ──────────→                   │
 │                                                            │
 │  Car:                                                      │
 │  1. Receive        → 󰤂 RF signal                          │
 │  2. Unpack payload → [512, 487, ...]                       │
 │  3. Extract drive  → throttle=512, steering=487            │
 │  4. Motor control  → Left: 50% forward, Right: 48% forward │
 │  5. Result         → 󰄀 Car turns slightly left            │
 │                                                            │
 └────────────────────────────────────────────────────────────┘
```

---

## Key Technical Details

### Payload Size
- **18 bytes total** (9 elements × 2 bytes per `int16_t`)
- Set via `radio.setPayloadSize(sizeof(payload))`

### Power Level
- `RF24_PA_LOW` - Reduces range but saves power and works for close-range operation

### Update Rate
- **100Hz** (every 10ms)
- Fast enough for responsive control
- Slow enough to avoid overwhelming the radio

### Button States (Currently Unused)
The controller sends button states (`payload[2]` through `payload[8]`), but the car currently **ignores** them. These could be used for:
- Start button: Enable/disable motors
- Select button: Switch driving modes
- D-pad: Control accessories (lights, horn, etc.)

### Future Enhancements
- 󰋜 Add bidirectional communication (car sends telemetry back)
- 󰄃 Implement button functionality
- 󰤂 Add automatic reconnection on signal loss
- 󰔡 Include battery voltage monitoring in payload
