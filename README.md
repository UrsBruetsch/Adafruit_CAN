# Adafruit_CAN Library – Function Reference 

Source: [github.com/adafruit/Adafruit_CAN](https://github.com/adafruit/Adafruit_CAN/tree/main/src)

## Overview

The library provides the `CANSAME5x` class for boards with a **native CAN peripheral** (e.g. the Adafruit Feather M4 CAN Express with an ATSAME51). This class inherits from `CANControllerClass` (the base from the well-known `arduino-CAN` API by Sandeep Mistry), which in turn derives from `Stream`/`Print`. This means both the CAN-specific functions and the familiar Arduino stream functions (`write`, `read`, `available`, `peek`, `flush`) are available.

**Include:**

```cpp
#include <CANSAME5x.h>
CANSAME5x CAN;
```

**Board-specific pins (Feather M4 CAN Express):**

```cpp
pinMode(PIN_CAN_STANDBY, OUTPUT);
digitalWrite(PIN_CAN_STANDBY, false); // turn off transceiver standby
pinMode(PIN_CAN_BOOSTEN, OUTPUT);
digitalWrite(PIN_CAN_BOOSTEN, true);  // turn on voltage booster
```

---

## Function Overview

| Function | Purpose |
| --- | --- |
| `begin(baudRate)` | Start the CAN bus at the given baud rate |
| `end()` | Stop the CAN bus |
| `beginPacket(id, dlc, rtr)` | Prepare a standard packet (11-bit ID) for sending |
| `beginExtendedPacket(id, dlc, rtr)` | Prepare an extended packet (29-bit ID) for sending |
| `endPacket()` | Send the prepared packet |
| `parsePacket()` | Check for an incoming packet |
| `packetId()` | ID of the most recently received packet |
| `packetExtended()` | Checks whether the received packet is extended |
| `packetRtr()` | Checks whether it's a Remote Transmission Request packet |
| `packetDlc()` | Requested data length for an RTR packet |
| `write(byte)` / `write(buffer, size)` | Write byte(s) into the current outgoing packet |
| `available()` | Number of bytes still readable in the received packet |
| `read()` | Read one byte from the received packet |
| `peek()` | Read a byte without advancing the read position |
| `flush()` | Stream interface requirement (no effect for CAN) |
| `onReceive(callback)` | Callback function for interrupt-driven reception |
| `filter(id, mask)` | Receive filter for standard IDs |
| `filterExtended(id, mask)` | Receive filter for extended IDs |
| `observe()` | Listen-only mode: bus is only monitored, no sending/ACK |
| `loopback()` | Internal loopback mode for testing |
| `sleep()` | Put the controller into sleep mode |
| `wakeup()` | Wake the controller from sleep mode |

---

## Functions in Detail with Examples

### `int begin(long baudRate)`

Initializes the CAN controller at the given baud rate (e.g. 250000 or 500000). Returns `1` on success, `0` on failure.

```cpp
if (!CAN.begin(250000)) {
  Serial.println("Starting CAN failed!");
  while (1) delay(10);
}
```

### `void end()`

Disables the CAN controller and releases the pins again.

```cpp
CAN.end();
```

### `int beginPacket(int id, int dlc = -1, bool rtr = false)`

Starts a standard packet (11-bit ID, 0–0x7FF). `dlc` is optional and used for RTR packets, `rtr` marks a Remote Transmission Request packet.

```cpp
CAN.beginPacket(0x12);
CAN.write('h');
CAN.write('i');
CAN.endPacket();
```

### `int beginExtendedPacket(long id, int dlc = -1, bool rtr = false)`

Like `beginPacket()`, but for extended IDs (29 bit, up to 0x1FFFFFFF).

```cpp
CAN.beginExtendedPacket(0xABCDEF);
CAN.write('h');
CAN.write('e');
CAN.write('l');
CAN.write('l');
CAN.write('o');
CAN.endPacket();
```

### `int endPacket()`

Finishes and sends the packet started with `beginPacket()`/`beginExtendedPacket()`. Returns `1` on success.

### `int parsePacket()`

Checks whether a new packet has been received and returns its data length (`0` if no packet is available). Must be called before `packetId()`, `read()`, etc.

```cpp
int packetSize = CAN.parsePacket();
if (packetSize) {
  Serial.print("Packet received, ID: 0x");
  Serial.println(CAN.packetId(), HEX);
}
```

### `long packetId()`

Returns the ID (standard or extended) of the packet most recently detected by `parsePacket()`.

### `bool packetExtended()`

`true` if the received packet uses an extended ID (29 bit).

### `bool packetRtr()`

`true` if the received packet is a Remote Transmission Request (contains no payload).

### `int packetDlc()`

For an RTR packet: the requested data length (Data Length Code).

```cpp
int packetSize = CAN.parsePacket();
if (packetSize) {
  if (CAN.packetRtr()) {
    Serial.print("RTR packet, requested length: ");
    Serial.println(CAN.packetDlc());
  } else {
    while (CAN.available()) {
      Serial.print((char)CAN.read());
    }
    Serial.println();
  }
}
```

### `size_t write(uint8_t byte)` / `size_t write(const uint8_t *buffer, size_t size)`

Writes a single byte or a buffer into the currently open outgoing packet (between `beginPacket()`/`beginExtendedPacket()` and `endPacket()`). Maximum 8 bytes per classic CAN packet.

```cpp
uint8_t data[4] = {0x01, 0x02, 0x03, 0x04};
CAN.beginPacket(0x100);
CAN.write(data, 4);
CAN.endPacket();
```

### `int available()`

Returns the number of unread bytes remaining in the current received packet.

### `int read()`

Reads the next byte from the current received packet (or `-1` if none is available).

### `int peek()`

Like `read()`, but does not advance the read position.

### `void flush()`

Present for compatibility with `Stream`; has essentially no effect for CAN.

### `void onReceive(void (*callback)(int))`

Registers a callback function that is invoked via interrupt for every received packet (parameter = packet size). An alternative to polling with `parsePacket()` in `loop()`.

```cpp
void onCanReceive(int packetSize) {
  Serial.print("Received, ID 0x");
  Serial.print(CAN.packetId(), HEX);
  Serial.print(", length ");
  Serial.println(packetSize);
  while (CAN.available()) {
    Serial.print((char)CAN.read());
  }
}

void setup() {
  // ... CAN.begin(...) ...
  CAN.onReceive(onCanReceive);
}
```

### `int filter(int id, int mask = 0x7ff)` / `int filterExtended(long id, long mask = 0x1fffffff)`

Sets a receive filter so that only packets with a matching ID (taking the mask into account) are passed through. `filter()` for standard IDs, `filterExtended()` for extended IDs.

```cpp
// Only receive packets with ID 0x120 (mask = compare all bits)
CAN.filter(0x120, 0x7ff);
```

### `int observe()`

Puts the controller into pure "listen-only" mode: the bus is only monitored, nothing is sent and no ACK is given. Useful for bus analysis.

```cpp
CAN.observe();
```

### `int loopback()`

Enables internal loopback mode: packets sent are immediately received again internally, without needing a physical bus or a second device. Handy for testing your own code.

```cpp
CAN.loopback();
CAN.beginPacket(0x01);
CAN.write(42);
CAN.endPacket();
// CAN.parsePacket() now returns the packet on the same device
```

### `int sleep()` / `int wakeup()`

Puts the controller into a low-power mode or wakes it back up.

```cpp
CAN.sleep();
// ... later ...
CAN.wakeup();
```

---

## Complete Example: Sender

```cpp
#include <CANSAME5x.h>
CANSAME5x CAN;

void setup() {
  Serial.begin(115200);
  while (!Serial) delay(10);

  pinMode(PIN_CAN_STANDBY, OUTPUT);
  digitalWrite(PIN_CAN_STANDBY, false);
  pinMode(PIN_CAN_BOOSTEN, OUTPUT);
  digitalWrite(PIN_CAN_BOOSTEN, true);

  if (!CAN.begin(250000)) {
    Serial.println("Starting failed!");
    while (1) delay(10);
  }
  Serial.println("CAN started");
}

void loop() {
  CAN.beginPacket(0x12);
  CAN.write('h');
  CAN.write('e');
  CAN.write('l');
  CAN.write('l');
  CAN.write('o');
  CAN.endPacket();
  delay(1000);
}
```

## Complete Example: Receiver (Polling)

```cpp
#include <CANSAME5x.h>
CANSAME5x CAN;

void setup() {
  Serial.begin(115200);
  while (!Serial) delay(10);

  pinMode(PIN_CAN_STANDBY, OUTPUT);
  digitalWrite(PIN_CAN_STANDBY, false);
  pinMode(PIN_CAN_BOOSTEN, OUTPUT);
  digitalWrite(PIN_CAN_BOOSTEN, true);

  if (!CAN.begin(250000)) {
    Serial.println("Starting failed!");
    while (1) delay(10);
  }
}

void loop() {
  int packetSize = CAN.parsePacket();
  if (packetSize) {
    Serial.print("Received ");
    if (CAN.packetExtended()) Serial.print("(extended) ");
    Serial.print("ID 0x");
    Serial.print(CAN.packetId(), HEX);
    Serial.print(": ");
    while (CAN.available()) {
      Serial.print((char)CAN.read());
    }
    Serial.println();
  }
}
```

---

## Notes

- The maximum payload per classic CAN packet is **8 bytes**.
- `filter()`/`filterExtended()` should be set before `begin()`, or right after starting it, before packets are processed.
- `onReceive()` is the interrupt-driven alternative to repeated `parsePacket()` polling in `loop()`.
- The concrete register-level details live in `CANSAME5x.cpp`/`CANSAME5x_port.h`; for plain application programming, the public interface listed above from `CANSAME5x.h` / `CANController.h` is sufficient.


