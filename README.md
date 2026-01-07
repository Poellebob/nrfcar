# NRF Car Communication Protocol

This document outlines the wireless communication protocol between the `nrfcontroller` (the remote) and the `nrfbot` (the car), which is facilitated by nRF24L01 radio modules.

## Overview

The communication is a simple, one-way link. The `nrfcontroller` acts as the **Transmitter (TX)**, continuously sending control data. The `nrfbot` acts as the **Receiver (RX)**, listening for this data to control its motors.

They communicate on two predefined addresses, or "pipes": `"1Node"` and `"2Node"`.

- The controller **writes** to the `"1Node"` address.
- The car **reads** from the `"1Node"` address.

## Controller (`nrfcontroller/src/main.cpp`)

The controller is configured to be the primary transmitter.

### Configuration

It sets its `radioNumber` to `0` and `role` to `true` (transmitter). This configures it to open a "writing pipe" to the address `"1Node"`.

```cpp
uint8_t address[][6] = {"1Node", "2Node"};
bool radioNumber = 0; // opposite of car
bool role = true;     // controller starts as TX

// ... in setup()
radio.openWritingPipe(address[radioNumber]);
radio.openReadingPipe(1, address[!radioNumber]);
// ...
if (role)
  radio.stopListening();
```

### Transmission Loop

In its main loop, the controller reads the state of its analog joysticks and digital buttons. This data is packed into a 9-element integer array called `payload` and sent out via the radio.

- `payload[0]`: X-axis of the analog stick (throttle)
- `payload[1]`: Y-axis of the analog stick (steering)
- `payload[2-8]`: State of the various digital buttons

```cpp
void loop() {
  payload[0] = analogRead(x_axis); // throttle
  payload[1] = analogRead(y_axis); // steering

  payload[2] = digitalRead(up_button);
  payload[3] = digitalRead(down_button);
  payload[4] = digitalRead(left_button);
  payload[5] = digitalRead(right_button);
  payload[6] = digitalRead(start_button);
  payload[7] = digitalRead(select_button);
  payload[8] = digitalRead(analog_button);

  radio.write(&payload, sizeof(payload));
  delay(10);
}
```

## Car (`nrfbot/src/main.cpp`)

The car is configured to be the primary receiver.

### Configuration

The car sets its `radioNumber` to `1` and `role` to `false` (receiver). This configures it to open a "reading pipe" on the `"1Node"` address, which is the same address the controller is writing to.

```cpp
uint8_t address[][6] = {"1Node", "2Node"};
bool radioNumber = 1; // this car uses address[1]
bool role = false;    // false = RX by default

// ... in setup()
radio.openWritingPipe(address[radioNumber]);
radio.openReadingPipe(1, address[!radioNumber]);
// ...
if (role)
  radio.stopListening();
else
  radio.startListening();
```

### Reception Loop

In its main loop, the car checks if any radio data is available. If it receives a packet, it reads the `payload` and uses the first two values to control the motors.

- `payload[0]` (throttle) and `payload[1]` (steering) are passed to the `self.drive()` function to move the car.

```cpp
void loop() {
  if (radio.available()) {
    radio.read(&payload, sizeof(payload));
    Serial.print(payload[0]);
    Serial.println(payload[1]);

    self.drive(payload[0], payload[1]);
  }
  delay(10);
}
```
