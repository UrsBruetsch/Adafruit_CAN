# Adafruit_CAN Library – Function Reference 

[Gehe zu Kapitel 1](#kapitel-1)

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




## Kapitel 1

# Adafruit_CAN Library – Funktionsreferenz

Quelle: [github.com/adafruit/Adafruit_CAN](https://github.com/adafruit/Adafruit_CAN/tree/main/src)

## Überblick

Die Library stellt für Boards mit **nativem CAN-Peripheriegerät** (z. B. Adafruit Feather M4 CAN Express mit ATSAME51) die Klasse `CANSAME5x` bereit. Diese erbt von `CANControllerClass` (Basis aus der bekannten `arduino-CAN`-API von Sandeep Mistry), die wiederum von `Stream`/`Print` ableitet. Dadurch stehen sowohl die CAN-spezifischen Funktionen als auch die gewohnten Arduino-Stream-Funktionen (`write`, `read`, `available`, `peek`, `flush`) zur Verfügung.

**Einbinden:**

```cpp
#include <CANSAME5x.h>
CANSAME5x CAN;
```

**Board-spezifische Pins (Feather M4 CAN Express):**

```cpp
pinMode(PIN_CAN_STANDBY, OUTPUT);
digitalWrite(PIN_CAN_STANDBY, false); // Standby des Transceivers ausschalten
pinMode(PIN_CAN_BOOSTEN, OUTPUT);
digitalWrite(PIN_CAN_BOOSTEN, true);  // Spannungsbooster einschalten
```

---

## Funktionsübersicht

| Funktion | Zweck |
| --- | --- |
| `begin(baudRate)` | CAN-Bus mit gegebener Baudrate starten |
| `end()` | CAN-Bus stoppen |
| `beginPacket(id, dlc, rtr)` | Standard-Paket (11-Bit-ID) zum Senden vorbereiten |
| `beginExtendedPacket(id, dlc, rtr)` | Extended-Paket (29-Bit-ID) zum Senden vorbereiten |
| `endPacket()` | Vorbereitetes Paket absenden |
| `parsePacket()` | Auf eingehendes Paket prüfen |
| `packetId()` | ID des zuletzt empfangenen Pakets |
| `packetExtended()` | Prüft, ob empfangenes Paket extended ist |
| `packetRtr()` | Prüft, ob Remote-Transmission-Request-Paket |
| `packetDlc()` | Angeforderte Datenlänge bei RTR-Paket |
| `write(byte)` / `write(buffer, size)` | Byte(s) in aktuelles Sendepaket schreiben |
| `available()` | Anzahl noch lesbarer Bytes im Empfangspaket |
| `read()` | Ein Byte aus Empfangspaket lesen |
| `peek()` | Byte lesen, ohne Leseposition zu bewegen |
| `flush()` | Stream-Pflichtfunktion (ohne Wirkung bei CAN) |
| `onReceive(callback)` | Callback-Funktion für interruptbasierten Empfang |
| `filter(id, mask)` | Empfangsfilter für Standard-IDs |
| `filterExtended(id, mask)` | Empfangsfilter für Extended-IDs |
| `observe()` | Bus nur mithören (Listen-only), kein Senden/ACK |
| `loopback()` | Interner Loopback-Modus zum Testen ohne Bus |
| `sleep()` | Controller in Sleep-Modus versetzen |
| `wakeup()` | Controller aus Sleep-Modus aufwecken |

---

## Funktionen im Detail mit Beispielen

### `int begin(long baudRate)`

Initialisiert den CAN-Controller mit der angegebenen Baudrate (z. B. 250000 oder 500000). Gibt `1` bei Erfolg, `0` bei Fehler zurück.

```cpp
if (!CAN.begin(250000)) {
  Serial.println("CAN-Start fehlgeschlagen!");
  while (1) delay(10);
}
```

### `void end()`

Deaktiviert den CAN-Controller und gibt die Pins wieder frei.

```cpp
CAN.end();
```

### `int beginPacket(int id, int dlc = -1, bool rtr = false)`

Startet ein Standard-Paket (11-Bit-ID, 0–0x7FF). `dlc` optional für RTR-Pakete, `rtr` markiert ein Remote-Transmission-Request-Paket.

```cpp
CAN.beginPacket(0x12);
CAN.write('h');
CAN.write('i');
CAN.endPacket();
```

### `int beginExtendedPacket(long id, int dlc = -1, bool rtr = false)`

Wie `beginPacket()`, jedoch für Extended-IDs (29 Bit, bis 0x1FFFFFFF).

```cpp
CAN.beginExtendedPacket(0xABCDEF);
CAN.write('h');
CAN.write('a');
CAN.write('l');
CAN.write('l');
CAN.write('o');
CAN.endPacket();
```

### `int endPacket()`

Schließt das mit `beginPacket()`/`beginExtendedPacket()` begonnene Paket ab und sendet es. Gibt `1` bei Erfolg zurück.

### `int parsePacket()`

Prüft, ob ein neues Paket empfangen wurde, und gibt dessen Datenlänge zurück (`0`, falls kein Paket vorliegt). Muss vor `packetId()`, `read()` usw. aufgerufen werden.

```cpp
int packetSize = CAN.parsePacket();
if (packetSize) {
  Serial.print("Paket empfangen, ID: 0x");
  Serial.println(CAN.packetId(), HEX);
}
```

### `long packetId()`

Liefert die ID (Standard oder Extended) des zuletzt mit `parsePacket()` erkannten Pakets.

### `bool packetExtended()`

`true`, wenn das empfangene Paket eine Extended-ID (29 Bit) verwendet.

### `bool packetRtr()`

`true`, wenn das empfangene Paket ein Remote-Transmission-Request ist (enthält keine Nutzdaten).

### `int packetDlc()`

Bei einem RTR-Paket: angeforderte Datenlänge (Data Length Code).

```cpp
int packetSize = CAN.parsePacket();
if (packetSize) {
  if (CAN.packetRtr()) {
    Serial.print("RTR-Paket, angeforderte Länge: ");
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

Schreibt ein einzelnes Byte bzw. einen Puffer in das aktuell offene Sendepaket (zwischen `beginPacket()`/`beginExtendedPacket()` und `endPacket()`). Maximal 8 Byte pro klassischem CAN-Paket.

```cpp
uint8_t daten[4] = {0x01, 0x02, 0x03, 0x04};
CAN.beginPacket(0x100);
CAN.write(daten, 4);
CAN.endPacket();
```

### `int available()`

Gibt die Anzahl der im aktuellen Empfangspaket noch nicht gelesenen Bytes zurück.

### `int read()`

Liest das nächste Byte aus dem aktuellen Empfangspaket (oder `-1`, wenn keins verfügbar ist).

### `int peek()`

Wie `read()`, rückt die Leseposition aber nicht weiter.

### `void flush()`

Aus Kompatibilität mit `Stream` vorhanden, hat bei CAN praktisch keine Wirkung.

### `void onReceive(void (*callback)(int))`

Registriert eine Callback-Funktion, die bei jedem empfangenen Paket per Interrupt aufgerufen wird (Parameter = Paketgröße). Alternative zum Polling mit `parsePacket()` in `loop()`.

```cpp
void onCanReceive(int packetSize) {
  Serial.print("Empfangen, ID 0x");
  Serial.print(CAN.packetId(), HEX);
  Serial.print(", Länge ");
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

Setzt einen Empfangsfilter, sodass nur Pakete mit passender ID (unter Berücksichtigung der Maske) durchgelassen werden. `filter()` für Standard-IDs, `filterExtended()` für Extended-IDs.

```cpp
// Nur Pakete mit ID 0x120 empfangen (Maske = alle Bits vergleichen)
CAN.filter(0x120, 0x7ff);
```

### `int observe()`

Versetzt den Controller in einen reinen "Mithör"-Modus (Listen-only): Der Bus wird nur beobachtet, es wird nicht gesendet und kein ACK gegeben. Nützlich zur Busanalyse.

```cpp
CAN.observe();
```

### `int loopback()`

Aktiviert den internen Loopback-Modus: Gesendete Pakete werden intern direkt wieder empfangen, ohne dass ein physischer Bus/zweites Gerät nötig ist. Praktisch zum Testen des eigenen Codes.

```cpp
CAN.loopback();
CAN.beginPacket(0x01);
CAN.write(42);
CAN.endPacket();
// CAN.parsePacket() liefert das Paket nun im selben Gerät zurück
```

### `int sleep()` / `int wakeup()`

Versetzt den Controller in einen Energiesparmodus bzw. weckt ihn wieder auf.

```cpp
CAN.sleep();
// ... später ...
CAN.wakeup();
```

---

## Vollständiges Beispiel: Sender

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
    Serial.println("Starten fehlgeschlagen!");
    while (1) delay(10);
  }
  Serial.println("CAN gestartet");
}

void loop() {
  CAN.beginPacket(0x12);
  CAN.write('h');
  CAN.write('a');
  CAN.write('l');
  CAN.write('l');
  CAN.write('o');
  CAN.endPacket();
  delay(1000);
}
```

## Vollständiges Beispiel: Empfänger (Polling)

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
    Serial.println("Starten fehlgeschlagen!");
    while (1) delay(10);
  }
}

void loop() {
  int packetSize = CAN.parsePacket();
  if (packetSize) {
    Serial.print("Empfangen ");
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

## Hinweise

- Die maximale Nutzlast pro klassischem CAN-Paket beträgt **8 Byte**.
- `filter()`/`filterExtended()` sollten vor `begin()` bzw. direkt nach dem Start gesetzt werden, bevor Pakete verarbeitet werden.
- `onReceive()` ist die interruptbasierte Alternative zu wiederholtem `parsePacket()`-Polling in `loop()`.
- Die konkreten Register-Details befinden sich in `CANSAME5x.cpp`/`CANSAME5x_port.h`; für die reine Anwendungsprogrammierung reicht die oben aufgeführte öffentliche Schnittstelle aus `CANSAME5x.h` / `CANController.h` aus.
