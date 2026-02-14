# ISA8 LPT COM RTC - Implementation Plan

## 1. ISA Bus Signals Used

| ISA Signal | Direction | Purpose |
|---|---|---|
| D0-D7 | Bidir | Data bus (active via 74245) |
| A0-A9 | In | Address bus (only A0-A9 needed, highest port 0x3F8 = 0b01_1111_1000) |
| AEN | In | DMA cycle indicator. Must be LOW for valid CPU I/O cycle |
| IOR# | In | I/O Read strobe (active low) |
| IOW# | In | I/O Write strobe (active low) |
| IRQ3-IRQ7 | Out | Interrupt request lines (directly active high, directly active low when no IRQ pending) |
| RESET DRV | In | System reset |

## 2. I/O Port Address Map

| Device | Option A (Default) | Option B | Address Range Size |
|---|---|---|---|
| UART1 (COM1) | 0x3F8-0x3FF | 0x2F8-0x2FF (COM2) | 8 bytes |
| UART2 (COM3) | 0x3E8-0x3EF | 0x2E8-0x2EF (COM4) | 8 bytes |
| PRN (LPT1) | 0x378-0x37F | 0x278-0x27F (LPT2) | 8 bytes |
| RTC | 0x70-0x71 | (fixed) | 2 bytes |

### Address Bit Patterns (A9..A0)

```
UART1 @ 0x3F8:  11 1111 1xxx    UART1 @ 0x2F8:  10 1111 1xxx
UART2 @ 0x3E8:  11 1110 1xxx    UART2 @ 0x2E8:  10 1110 1xxx
PRN   @ 0x378:  01 1011 1xxx    PRN   @ 0x278:  00 1001 11xxx
RTC   @ 0x070:  00 0111 000x    (fixed, no jumper)
```

Note: `x` bits are don't-care for chip-select (directly decoded by each device's A0-A2 or internal addressing).

## 3. Jumper Design

### 3.1 Jumper Inputs to ATF22V10

Instead of feeding raw address lines and muxing them inside the PLD, use jumpers to route one of two address lines to a single PLD input. This minimizes PLD pin usage.

The key observation: for each device pair, only **one address bit** changes between Option A and Option B:

| Device | Option A | Option B | Differing Bit |
|---|---|---|---|
| UART1 | 0x3F8 (A9=1) | 0x2F8 (A9=0) | A9 |
| UART2 | 0x3E8 (A9=1) | 0x2E8 (A9=0) | A9 |
| PRN | 0x378 (A9=0, A8=1) | 0x278 (A9=0, A8=0) | A8 |

However, UART1 and UART2 share the same differing bit (A9), so a single jumper can't serve both if they need independent selection. But in practice, UART1 at 0x3F8/0x2F8 and UART2 at 0x3E8/0x2E8 can share the A9 jumper only if they are always switched together. The spec implies independent configuration, so we need separate jumper inputs.

**Revised approach: one jumper bit per device fed into the PLD.**

- **J_UART1**: 1-bit jumper (HIGH = 0x3F8, LOW = 0x2F8)
- **J_UART2**: 1-bit jumper (HIGH = 0x3E8, LOW = 0x2E8)
- **J_PRN**: 1-bit jumper (HIGH = 0x378, LOW = 0x278)

These jumper signals connect directly to ATF22V10 inputs. The PLD equations incorporate them.

### 3.2 Device Enable/Disable Jumpers

To disable a device completely (nice-to-have from spec):

- **EN_UART1**: jumper, when open (pulled HIGH via resistor) = disabled. When shorted to GND = enabled.
- **EN_UART2**: same scheme.
- **EN_PRN**: same scheme.

These also go into the ATF22V10 as inputs. When the enable input is HIGH (disabled), the PLD simply never asserts the corresponding CS output.

### 3.3 IRQ Selection Jumpers

IRQ routing does NOT go through the PLD. Use simple 3-pin jumper headers connecting each device's IRQ output to the selected ISA IRQ line.

| Jumper | Position 1 | Position 2 | Notes |
|---|---|---|---|
| J_IRQ_UART1 | IRQ4 | IRQ3 | Standard COM1=IRQ4, COM2=IRQ3 |
| J_IRQ_UART2 | IRQ4 | IRQ3 | Standard COM3=IRQ4, COM4=IRQ3 |
| J_IRQ_PRN | IRQ7 | IRQ5 | Standard LPT1=IRQ7, LPT2=IRQ5 |
| J_IRQ_RTC | IRQ7 / IRQ6 / IRQ5 | — | 4-pin header, jumper selects one of 3 |

For the RTC IRQ (3 choices: IRQ5, IRQ6, IRQ7), use a 1x4 pin header:
```
Pin 1: RTC_IRQ (from DS12885 IRQ pin, accent via open-drain pullup)
Pin 2: IRQ5
Pin 3: IRQ6
Pin 4: IRQ7
```
A single jumper wire connects Pin 1 to one of Pins 2-4.

### 3.4 Jumper Summary Table

| Jumper | Type | Pins | Function |
|---|---|---|---|
| JP_UART1_ADDR | 3-pin header | 1=A9, 2=UART1_SEL, 3=VCC | Address: center-to-1 = 0x3F8, center-to-3 = 0x2F8 |
| JP_UART1_EN | 2-pin header | 1=UART1_EN, 2=GND | Shorted = enabled, open = disabled (pull-up) |
| JP_UART1_IRQ | 3-pin header | 1=IRQ4, 2=UART1_IRQ, 3=IRQ3 | Select IRQ |
| JP_UART2_ADDR | 3-pin header | 1=A9, 2=UART2_SEL, 3=VCC | Address: center-to-1 = 0x3E8, center-to-3 = 0x2E8 |
| JP_UART2_EN | 2-pin header | 1=UART2_EN, 2=GND | Shorted = enabled, open = disabled (pull-up) |
| JP_UART2_IRQ | 3-pin header | 1=IRQ4, 2=UART2_IRQ, 3=IRQ3 | Select IRQ |
| JP_PRN_ADDR | 3-pin header | 1=A8, 2=PRN_SEL, 3=VCC | Address: center-to-1 = 0x378, center-to-3 = 0x278 |
| JP_PRN_EN | 2-pin header | 1=PRN_EN, 2=GND | Shorted = enabled, open = disabled (pull-up) |
| JP_PRN_IRQ | 3-pin header | 1=IRQ7, 2=PRN_IRQ, 3=IRQ5 | Select IRQ |
| JP_RTC_IRQ | 1x4 header | 1=RTC_IRQ, 2=IRQ5, 3=IRQ6, 4=IRQ7 | Jumper wire Pin1 to one of Pin2-4 |

**Note on address jumper wiring (example JP_UART1_ADDR):**
```
  Pin 1 ---- A9 (from ISA bus)
  Pin 2 ---- UART1_SEL (to ATF22V10 input)
  Pin 3 ---- VCC (+5V)
```
- Jumper on Pin1-Pin2: UART1_SEL follows A9 → PLD sees A9 → matches 0x3F8
- Jumper on Pin2-Pin3: UART1_SEL = VCC (always HIGH) → PLD treats this as A9=1 equivalent

Wait - this needs correction. Let me reconsider:

For UART1: 0x3F8 has A9=1, 0x2F8 has A9=0. The PLD needs to match "A9=1" for 0x3F8 or "A9=0" for 0x2F8.

Better approach - feed the jumper output directly as a "configuration bit" into the PLD, and also feed the raw A9 into the PLD. The PLD equation uses XNOR(A9, J_UART1) to check if A9 matches the expected value:
- J_UART1 = 1 (jumper to VCC): PLD expects A9=1 → 0x3F8
- J_UART1 = 0 (jumper to GND): PLD expects A9=0 → 0x2F8

This is cleaner. Use simple 2-pin jumpers:

| Jumper | Shorted (to VCC) | Open (pull-down to GND) |
|---|---|---|
| J_UART1_ADDR | 0x3F8 (A9 must be 1) | 0x2F8 (A9 must be 0) |
| J_UART2_ADDR | 0x3E8 (A9 must be 1) | 0x2E8 (A9 must be 0) |
| J_PRN_ADDR | 0x378 (A8 must be 1) | 0x278 (A8 must be 0) |

Each is a 2-pin header with a pull-down resistor (10k to GND). Jumper cap shorts to VCC for the "A" option.

Alternatively, use standard 3-pin jumpers (VCC / signal / GND):

```
  Pin 1 = VCC
  Pin 2 = J_xxx (to PLD input, weak pull-down to GND via 10k)
  Pin 3 = GND
```
- Cap on 1-2: signal = HIGH → Option A
- Cap on 2-3: signal = LOW → Option B
- No cap: signal = LOW (pull-down) → Option B (default)

## 4. ATF22V10 Pin Assignment and Address Decoding

### 4.1 ATF22V10C Pinout (24-pin DIP/PLCC)

- Pins 1, 2-11, 13: Inputs (pin 1 is clock/input, pins 2-11 are dedicated inputs, pin 13 is dedicated input)
- Pins 14-23: I/O (can be output or feedback)
- Pin 12: GND
- Pin 24: VCC

### 4.2 Pin Assignment

| ATF22V10 Pin | Direction | Signal | Purpose |
|---|---|---|---|
| 1 | Input | AEN | DMA indicator (active high = DMA, must be 0 for I/O) |
| 2 | Input | A0 | Address bit 0 (for RTC decode) |
| 3 | Input | A3 | Address bit 3 |
| 4 | Input | A4 | Address bit 4 |
| 5 | Input | A5 | Address bit 5 |
| 6 | Input | A6 | Address bit 6 |
| 7 | Input | A7 | Address bit 7 |
| 8 | Input | A8 | Address bit 8 |
| 9 | Input | A9 | Address bit 9 |
| 10 | Input | J_UART1 | Jumper: UART1 address select (1=0x3F8, 0=0x2F8) |
| 11 | Input | J_UART2 | Jumper: UART2 address select (1=0x3E8, 0=0x2E8) |
| 12 | — | GND | Ground |
| 13 | Input | J_PRN | Jumper: PRN address select (1=0x378, 0=0x278) |
| 14 | Output | UART1_CS# | UART1 chip select (active low) |
| 15 | Output | UART2_CS# | UART2 chip select (active low) |
| 16 | Output | PRN_CS# | PRN chip select (active low) |
| 17 | Output | RTC_CS# | RTC chip select (active low) |
| 18 | Output | BUF_EN# | 74245 enable (active low) |
| 19 | Output | (spare) | |
| 20 | Input | EN_PRN# | PRN enable jumper (0=enabled, active when shorted to GND) |
| 21 | Input | EN_UART2# | UART2 enable jumper (0=enabled) |
| 22 | Input | EN_UART1# | UART1 enable jumper (0=enabled) |
| 23 | Output | (spare) | |
| 24 | — | VCC | +5V |

**Note:** A1 and A2 are not needed for chip-select decode — they are decoded internally by each peripheral (UART uses A0-A2, PRN uses A0-A2). We only need A0 for the RTC (which uses port 0x70 and 0x71, differing by A0).

### 4.3 Address Decoding Equations (Active-Low Outputs)

For each device, the chip-select is active when:
1. AEN = 0 (not a DMA cycle)
2. The address bits match the selected port range
3. The device is enabled (EN_xxx# = 0)

#### UART1_CS# (active low)

```
Port 0x3F8 when J_UART1=1:  A9=1, A8=1, A7=1, A6=1, A5=1, A4=1, A3=1  (0b11_1111_1xxx)
Port 0x2F8 when J_UART1=0:  A9=0, A8=1, A7=1, A6=1, A5=1, A4=1, A3=1  (0b10_1111_1xxx)

UART1_CS# = !( !AEN & !EN_UART1# & (A9 XNOR J_UART1) & A8 & A7 & A6 & A5 & A4 & A3 )

Expanded (sum-of-products):
UART1_CS# = !( !AEN & !EN_UART1# & A9 & J_UART1 & A8 & A7 & A6 & A5 & A4 & A3
             + !AEN & !EN_UART1# & !A9 & !J_UART1 & A8 & A7 & A6 & A5 & A4 & A3 )
```

#### UART2_CS# (active low)

```
Port 0x3E8 when J_UART2=1:  A9=1, A8=1, A7=1, A6=1, A5=1, A4=0, A3=1  (0b11_1110_1xxx)
Port 0x2E8 when J_UART2=0:  A9=0, A8=1, A7=1, A6=1, A5=1, A4=0, A3=1  (0b10_1110_1xxx)

UART2_CS# = !( !AEN & !EN_UART2# & (A9 XNOR J_UART2) & A8 & A7 & A6 & A5 & !A4 & A3 )
```

#### PRN_CS# (active low)

```
Port 0x378 when J_PRN=1:  A9=0, A8=1, A7=1, A6=0, A5=1, A4=1, A3=1  (0b01_1011_1xxx)
Port 0x278 when J_PRN=0:  A9=0, A8=0, A7=1, A6=0, A5=1, A4=1, A3=1  (0b00_1001_11xxx)

Wait — let me recheck:
0x378 = 0011 0111 1000 → A9=0 A8=1 A7=1 A6=0 A5=1 A4=1 A3=1 A2=0 A1=0 A0=0 (base)
0x278 = 0010 0111 1000 → A9=0 A8=0 A7=1 A6=0 A5=1 A4=1 A3=1

Differing bit: A8

PRN_CS# = !( !AEN & !EN_PRN# & !A9 & (A8 XNOR J_PRN) & A7 & !A6 & A5 & A4 & A3 )
```

#### RTC_CS# (active low)

```
Port 0x70-0x71:  A9=0, A8=0, A7=0, A6=1, A5=1, A4=1, A3=0  (0b00_0111_000x)

RTC_CS# = !( !AEN & !A9 & !A8 & !A7 & A6 & A5 & A4 & !A3 )
```
RTC has no enable jumper and no address jumper (fixed at 0x70-0x71).

#### BUF_EN# (74245 active-low enable)

The bus buffer must be active whenever any device on the card is being accessed:

```
BUF_EN# = UART1_CS# & UART2_CS# & PRN_CS# & RTC_CS#

(i.e., BUF_EN# goes low when any CS# goes low)

Equivalently:
BUF_EN# = !( !UART1_CS# + !UART2_CS# + !PRN_CS# + !RTC_CS# )
         = !( !UART1_CS# | !UART2_CS# | !PRN_CS# | !RTC_CS# )
```

In the PLD, this is simply the OR of all chip-select conditions, then inverted output.

### 4.4 CUPL Source Skeleton

```cupl
Name     ISA8_DECODE;
PartNo   01;
Date     2026-02-14;
Designer VK;
Company  ;
Assembly ;
Location ;
Device   g22v10;

/* Inputs */
Pin 1  = AEN;
Pin 2  = A0;
Pin 3  = A3;
Pin 4  = A4;
Pin 5  = A5;
Pin 6  = A6;
Pin 7  = A7;
Pin 8  = A8;
Pin 9  = A9;
Pin 10 = J_UART1;
Pin 11 = J_UART2;
Pin 13 = J_PRN;

/* Active-low enable jumpers (active when LOW = shorted to GND) */
Pin 22 = !EN_UART1;  /* active low input */
Pin 21 = !EN_UART2;
Pin 20 = !EN_PRN;

/* Outputs (directly active-low; directly active-low directly active-low directly active low) */
Pin 14 = !UART1_CS;
Pin 15 = !UART2_CS;
Pin 16 = !PRN_CS;
Pin 17 = !RTC_CS;
Pin 18 = !BUF_EN;

/* Intermediate terms */
FIELD addr = [A9,A8,A7,A6,A5,A4,A3];

/* UART1: 0x3F8 (A9..A3 = 1111111) or 0x2F8 (A9..A3 = 0111111) */
UART1_CS = !AEN & EN_UART1
         & ( A9 &  J_UART1 # !A9 & !J_UART1)
         & A8 & A7 & A6 & A5 & A4 & A3;

/* UART2: 0x3E8 (A9..A3 = 1111011) or 0x2E8 (A9..A3 = 0111011) */  /* Corrected: A9..A3 = x111 0 1 1 → wait */
/* 0x3E8 = 11 1110 1000  → A9=1 A8=1 A7=1 A6=1 A5=1 A4=0 A3=1 */
/* 0x2E8 = 10 1110 1000  → A9=0 A8=1 A7=1 A6=1 A5=1 A4=0 A3=1 */
UART2_CS = !AEN & EN_UART2
         & ( A9 &  J_UART2 # !A9 & !J_UART2)
         & A8 & A7 & A6 & A5 & !A4 & A3;

/* PRN: 0x378 (A9..A3 = 0110111) or 0x278 (A9..A3 = 0010111) */
/* 0x378 = 01 1011 1000  → A9=0 A8=1 A7=1 A6=0 A5=1 A4=1 A3=1 */
/* 0x278 = 00 1001 11000 → wait, 0x278 = 10 0111 1000 → A9=0 A8=0 A7=1 A6=0 A5=1 A4=1 A3=1 */
/* Correction: 0x278 = 0010_0111_1000 → only 10 bits: */
/*   bit9=0 bit8=1 bit7=0 bit6=0 bit5=1 bit4=1 bit3=1 bit2=0 bit1=0 bit0=0 */
/* Wait, let me redo this carefully: */
/*   0x278 = 632 decimal. 632 = 512+64+32+16+8 = 0b10_0111_1000 */
/*   A9=1? No. 0x278 = 2*256+7*16+8 = 512+112+8 = 632 */
/*   632 = 1001111000 binary (10 bits) → A9=1, A8=0, A7=0, A6=1, A5=1, A4=1, A3=1, A2=0, A1=0, A0=0 */
/* That doesn't look right either. Let me be very careful: */
/*   0x278 hex = 2*16^2 + 7*16 + 8 = 512 + 112 + 8 = 632 */
/*   632 in binary: 512=2^9, 632-512=120, 64=2^6, 120-64=56, 32=2^5, 56-32=24, 16=2^4, 24-16=8, 8=2^3 */
/*   So: 2^9 + 2^6 + 2^5 + 2^4 + 2^3 = 10_0111_1000 */
/*   A9=1 A8=0 A7=0 A6=1 A5=1 A4=1 A3=1 */

/* Hmm, but 0x378: 3*256+7*16+8 = 768+112+8 = 888 */
/*   888 = 512+256+64+32+16+8 = 2^9+2^8+2^6+2^5+2^4+2^3 */
/*   = 11_0111_1000 */
/*   A9=1 A8=1 A7=0 A6=1 A5=1 A4=1 A3=1 */

/* So the differing bit between 0x378 and 0x278 is A8 (1 vs 0). A9=1 for both! */

PRN_CS = !AEN & EN_PRN
       & A9 & (A8 XNOR J_PRN) & !A7 & A6 & A5 & A4 & A3;

/* RTC: 0x70 = 0000_0111_0000 (10 bits: 00_0111_000x) */
/*   0x70 = 112 = 64+32+16 = 2^6+2^5+2^4 */
/*   A9=0 A8=0 A7=0 A6=1 A5=1 A4=1 A3=0 */
RTC_CS = !AEN & !A9 & !A8 & !A7 & A6 & A5 & A4 & !A3;

/* Buffer enable: active when any CS is active */
BUF_EN = UART1_CS # UART2_CS # PRN_CS # RTC_CS;
```

## 5. DS12885 RTC Interface to ISA Bus

### 5.1 Bus Mode Selection

Intel bus timing mode is selected by leaving MOT pin unconnected (internal 20k pull-down to GND).

### 5.2 Signal Connections

| DS12885 Pin | Signal | ISA / Board Connection |
|---|---|---|
| MOT | Bus mode | Leave unconnected (pulled low internally → Intel mode) |
| AD0-AD7 | Mux Addr/Data | Connected to card-side data bus (74245 B-side). These are shared address AND data lines. |
| CS | Chip select | RTC_CS# from ATF22V10 (directly active low already) |
| AS | Address strobe | Active-high pulse, directly connected to address latch — see below |
| DS | Data strobe / OE | Connected to IOR# (active low in Intel mode — acts as output enable for reads) |
| R/W | Read/Write | Connected to IOW# (active low in Intel mode — acts as write enable; data latched on rising edge) |
| RESET | Reset | Connected to RESET DRV from ISA bus |
| IRQ | Interrupt out | Open-drain, connected via jumper to IRQ5/6/7, needs 4.7k pull-up to VCC |
| SQW | Square wave | Leave unconnected or route to test point |
| X1, X2 | Crystal | 32.768 kHz crystal (Y1) |
| VCC | Power | +5V from ISA bus |
| VBAT | Backup battery | CR2032 (3V) via battery holder |
| RCLR | RAM clear | Tie to VCC via 10k pull-up (or add a push-button to GND for manual RAM clear) |

### 5.3 Multiplexed Address/Data Bus — Address Latch Mechanism

The DS12885 uses a **multiplexed** address/data bus (AD0-AD7). The CPU must first put the register address on the bus, strobe AS, and then perform the data transfer. In a standard IBM AT, the RTC is accessed at ports 0x70 (address register) and 0x71 (data register):

- **Write to port 0x70**: Latches the RTC internal register address (written data goes to the RTC's address latch). The DS12885 expects AS to pulse high, then the data appears as the "address phase."
- **Read/Write to port 0x71**: Accesses the previously addressed RTC register data.

**Key insight**: The DS12885's AS (Address Strobe) pin must be pulsed HIGH when the CPU writes to port 0x70, to latch the address. When port 0x71 is accessed, AS stays LOW, and the bus cycle is a normal data read/write.

**Implementation**: Use **A0** (which is 0 for port 0x70 and 1 for port 0x71) to generate AS:

```
AS = !A0 & RTC_CS_active & IOW_active
```

More precisely, AS should go HIGH when:
- The CPU is writing to port 0x70 (A0=0)
- RTC_CS# is asserted

Since in Intel mode the DS12885 expects:
- AS rising edge latches the address from AD0-AD7
- Then DS (connected to IOR#) or R/W (connected to IOW#) performs the data transfer

The simplest approach used in IBM AT designs:

**AS = NOT(A0)**  (directly invert A0, active high when A0=0)

This is gated by CS already (the RTC ignores AS when CS is not asserted). During a write to 0x70, A0=0, so AS=1, and the data on AD0-AD7 is latched as the register address. During access to 0x71, A0=1, so AS=0, and the data transfer occurs normally.

**Implementation**: Use a single inverter gate (one section of a 74HC04) or a spare gate:

```
RTC_AS = !A0
```

Connect this directly to the DS12885 AS pin. The RTC's CS pin ensures this only takes effect during RTC cycles.

### 5.4 Intel Mode Timing Summary

For a **write to port 0x70** (set RTC register address):
1. CPU puts 0x70 on address bus, AEN=0
2. ATF22V10 asserts RTC_CS# low, BUF_EN# low
3. AS = !A0 = 1 (A0=0 for port 0x70)
4. CPU drives data (register address 0x00-0x7F) on D0-D7 → through 74245 → AD0-AD7
5. IOW# goes low then high → DS12885 latches address from AD0-AD7 on IOW# rising edge
6. AS returns low when bus cycle ends

For a **read from port 0x71** (read RTC register data):
1. CPU puts 0x71 on address bus, AEN=0
2. ATF22V10 asserts RTC_CS# low, BUF_EN# low
3. AS = !A0 = 0 (A0=1 for port 0x71) — no address latch, data phase
4. IOR# goes low → DS12885 DS pin goes low → RTC drives data on AD0-AD7
5. 74245 direction = B→A (IOR# = 0 → DIR = 0) → data flows to ISA D0-D7
6. CPU reads data

For a **write to port 0x71** (write RTC register data):
1. Same as read but IOW# goes low instead
2. DS12885 R/W pin (connected to IOW#) goes low then high → data latched on rising edge

## 6. 74245 Bus Buffer

### 6.1 Connections

| 74245 Pin | Signal | Connection |
|---|---|---|
| A1-A8 (pins 2-9) | ISA D0-D7 | ISA bus data lines |
| B1-B8 (pins 11-18) | Card D0-D7 | UART1, UART2, PRN data pins, DS12885 AD0-AD7 |
| DIR (pin 1) | Direction | Connected to IOR# from ISA bus |
| G# (pin 19) | Enable | BUF_EN# from ATF22V10 |

### 6.2 Direction Logic

- IOR# = 0 (I/O read active): DIR = 0 → B-to-A (device → ISA bus)
- IOR# = 1 (idle or I/O write): DIR = 1 → A-to-B (ISA bus → device)

### 6.3 Enable Logic

G# (BUF_EN#) is active low. The 74245 is enabled only when a device on this card is being addressed, preventing bus conflicts with other ISA cards.

## 7. Corrected Address Bit Analysis

Let me present the final, verified address patterns:

```
Address  Hex    Binary (A9 A8 A7 A6 A5 A4 A3 A2 A1 A0)
0x070    112    0  0  0  1  1  1  0  0  0  0    RTC addr register
0x071    113    0  0  0  1  1  1  0  0  0  1    RTC data register
0x278    632    1  0  0  1  1  1  1  0  0  0    LPT2 base
0x2E8    744    1  0  1  1  1  0  1  0  0  0    COM4 base
0x2F8    760    1  0  1  1  1  1  1  0  0  0    COM2 base
0x378    888    1  1  0  1  1  1  1  0  0  0    LPT1 base
0x3E8   1000    1  1  1  1  1  0  1  0  0  0    COM3 base
0x3F8   1016    1  1  1  1  1  1  1  0  0  0    COM1 base
```

### Corrected Decode Patterns (A9..A3 only, A2-A0 decoded by devices)

| Device | Port | A9 | A8 | A7 | A6 | A5 | A4 | A3 |
|---|---|---|---|---|---|---|---|---|
| UART1=0x3F8 | COM1 | 1 | 1 | 1 | 1 | 1 | 1 | 1 |
| UART1=0x2F8 | COM2 | 1 | 0 | 1 | 1 | 1 | 1 | 1 |
| UART2=0x3E8 | COM3 | 1 | 1 | 1 | 1 | 1 | 0 | 1 |
| UART2=0x2E8 | COM4 | 1 | 0 | 1 | 1 | 1 | 0 | 1 |
| PRN=0x378 | LPT1 | 1 | 1 | 0 | 1 | 1 | 1 | 1 |
| PRN=0x278 | LPT2 | 1 | 0 | 0 | 1 | 1 | 1 | 1 |
| RTC=0x070 | RTC | 0 | 0 | 0 | 1 | 1 | 1 | 0 |

### Corrected Observations

- **UART1**: 0x3F8 vs 0x2F8 differ in **A8** (not A9 as originally assumed!)
- **UART2**: 0x3E8 vs 0x2E8 differ in **A8**
- **PRN**: 0x378 vs 0x278 differ in **A8**
- **All three** devices use A8 as the switching bit! A9=1 for all UART/PRN ports.

### Corrected Jumper Scheme

Since all three devices switch on A8, we can simplify:

| Jumper | Controls | HIGH (shorted to VCC) | LOW (to GND) |
|---|---|---|---|
| J_UART1 | A8 match for UART1 | 0x3F8 (A8=1) | 0x2F8 (A8=0) |
| J_UART2 | A8 match for UART2 | 0x3E8 (A8=1) | 0x2E8 (A8=0) |
| J_PRN | A8 match for PRN | 0x378 (A8=1) | 0x278 (A8=0) |

### Corrected CUPL Equations

```cupl
/* UART1: 0x3F8 (A9..A3 = 1111111) or 0x2F8 (A9..A3 = 1011111) */
/* Differ in A8. Common: A9=1, A7=1, A6=1, A5=1, A4=1, A3=1 */
UART1_CS = !AEN & EN_UART1
         & A9 & (A8 $ !J_UART1) & A7 & A6 & A5 & A4 & A3;
/* A8 $ !J_UART1 is XNOR(A8, J_UART1): true when A8 matches J_UART1 */
/* In CUPL: XNOR is written as !(A8 $ J_UART1) or equivalently (A8 $ !J_UART1) */

/* UART2: 0x3E8 (A9..A3 = 1111 0 1 1) or 0x2E8 (A9..A3 = 1011 0 1 1) */
/* Differ in A8. Common: A9=1, A7=1, A6=1, A5=1, A4=0, A3=1 */
UART2_CS = !AEN & EN_UART2
         & A9 & (A8 $ !J_UART2) & A7 & A6 & A5 & !A4 & A3;

/* PRN: 0x378 (A9..A3 = 1101111) or 0x278 (A9..A3 = 1001111) */
/* Differ in A8. Common: A9=1, A7=0, A6=1, A5=1, A4=1, A3=1 */
PRN_CS = !AEN & EN_PRN
       & A9 & (A8 $ !J_PRN) & !A7 & A6 & A5 & A4 & A3;

/* RTC: 0x70 = A9..A3 = 0001110 */
RTC_CS = !AEN & !A9 & !A8 & !A7 & A6 & A5 & A4 & !A3;

/* Buffer enable */
BUF_EN = UART1_CS # UART2_CS # PRN_CS # RTC_CS;
```

## 8. Complete ATF22V10 Pin Assignment (Corrected)

| ATF22V10 Pin | Direction | Signal | Purpose |
|---|---|---|---|
| 1 | Input | AEN | DMA cycle indicator |
| 2 | Input | A0 | Address bit 0 (for RTC AS generation, directly active-low directly active low directly active low directly) |
| 3 | Input | A3 | Address bit 3 |
| 4 | Input | A4 | Address bit 4 |
| 5 | Input | A5 | Address bit 5 |
| 6 | Input | A6 | Address bit 6 |
| 7 | Input | A7 | Address bit 7 |
| 8 | Input | A8 | Address bit 8 (key switching bit) |
| 9 | Input | A9 | Address bit 9 |
| 10 | Input | J_UART1 | Jumper: UART1 address select |
| 11 | Input | J_UART2 | Jumper: UART2 address select |
| 12 | — | GND | Ground |
| 13 | Input | J_PRN | Jumper: PRN address select |
| 14 | Output | UART1_CS# | UART1 chip select (directly active low) |
| 15 | Output | UART2_CS# | UART2 chip select (directly active low) |
| 16 | Output | PRN_CS# | PRN chip select (directly active low) |
| 17 | Output | RTC_CS# | RTC chip select (directly active low) |
| 18 | Output | BUF_EN# | 74245 output enable (directly active low) |
| 19 | Output | RTC_AS | DS12885 Address Strobe (active high = !A0, directly active high when A0=0) |
| 20 | Input | EN_PRN# | PRN enable jumper (pulled up, short to GND to enable) |
| 21 | Input | EN_UART2# | UART2 enable jumper |
| 22 | Input | EN_UART1# | UART1 enable jumper |
| 23 | Output | (spare) | Reserved for future use |
| 24 | — | VCC | +5V |

### RTC_AS Equation (added to PLD)

```cupl
/* RTC Address Strobe: active high when A0=0 during RTC access */
/* In IBM AT compatible design, AS simply inverts A0 */
/* CS ensures the DS12885 ignores AS when not selected */
RTC_AS = !A0;
```

Note: RTC_AS could also be generated with a simple inverter outside the PLD, but using a PLD output saves a chip.

## 9. Bill of Materials (Logic Section)

| Ref | Component | Package | Qty | Purpose |
|---|---|---|---|---|
| U1 | ATF22V10C-15PU | DIP-24 | 1 | Address decode PLD |
| U2 | 74HCT245 | DIP-20 | 1 | Bidirectional bus buffer |
| U3 | DS12885 | DIP-24 / PLCC-28 | 1 | Real-time clock |
| U4 | 16C450 or 16C550 | DIP-40 | 2 | UART (COM1, COM2) |
| U5 | UM82C11-C | DIP-40 | 1 | Parallel port controller |
| U6 | GD75232N | DIP-20 | 1-2 | RS-232 level shifter |
| Y1 | 32.768 kHz crystal | HC49 | 1 | RTC oscillator |
| Y2 | 1.8432 MHz oscillator | DIP-8/14 | 1 | UART clock |
| BT1 | CR2032 holder | Through-hole | 1 | RTC battery backup |
| R1-R3 | 10k resistor | Axial | 3 | Pull-up for EN_xxx# jumpers |
| R4 | 4.7k resistor | Axial | 1 | Pull-up for DS12885 IRQ (open-drain) |
| R5-R7 | 10k resistor | Axial | 3 | Pull-down for J_xxx addr jumpers |
| JP1-JP10 | Pin headers | 2.54mm | Various | Jumper configuration headers |

## 10. Schematic Block Diagram

```
                    ISA 8-bit Bus
                    ═══════════════════════════════════
                    │D0-D7  A0-A9  IOR# IOW# AEN  IRQ3-7  RESET│
                    │   │     │      │    │    │      │       │  │
                    │   │     │      │    │    │      │       │  │
                ┌───┼───┼─────┼──────┼────┼────┼──────┼───────┼──┤
                │   │   │     │      │    │    │      │       │  │
    ┌───────────┤   │   │     │      │    │    │      │       │  │
    │  74HCT245 │   │   │     │      │    │    │      │       │  │
    │  ┌────┐   │   │   │     │      │    │    │      │       │  │
    │  │A  B│◄──┼───┘   │     │      │    │    │      │       │  │
    │  │    │───┼──┐    │     │      │    │    │      │       │  │
    │  │DIR │◄──┼──┼────┼─────┼──────┘    │    │      │       │  │
    │  │ G# │◄──┼──┼──┐ │     │            │    │      │       │  │
    │  └────┘   │  │  │ │     │            │    │      │       │  │
    └───────────┘  │  │ │     │            │    │      │       │  │
                   │  │ │     │            │    │      │       │  │
    ┌──────────────┼──┼─┼─────┼────────────┼────┼──────┼───────┼──┤
    │ ATF22V10     │  │ │     │            │    │      │       │  │
    │              │  │ │     │            │    │      │       │  │
    │ A0-A9,AEN ◄──┼──┼─┼─────┘            │    │      │       │  │
    │ J_UART1/2 ◄──┼──┼─┼── Jumpers        │    │      │       │  │
    │ J_PRN     ◄──┼──┼─┼── Jumpers        │    │      │       │  │
    │ EN_*#     ◄──┼──┼─┼── Jumpers        │    │      │       │  │
    │              │  │ │                   │    │      │       │  │
    │ UART1_CS# ───┼──┼─┼──► UART1 CS2     │    │      │       │  │
    │ UART2_CS# ───┼──┼─┼──► UART2 CS2     │    │      │       │  │
    │ PRN_CS#   ───┼──┼─┼──► PRN CS        │    │      │       │  │
    │ RTC_CS#   ───┼──┼─┼──► DS12885 CS    │    │      │       │  │
    │ BUF_EN#   ───┼──┘ │                  │    │      │       │  │
    │ RTC_AS    ───┼────►│  DS12885 AS     │    │      │       │  │
    └──────────────┘     │                  │    │      │       │  │
                         │                  │    │      │       │  │
                Card Data Bus (from 74245 B-side)│      │       │  │
                   ══════╪══════════════════╪════╪══════╪═══════╪══╪
                         │                  │    │      │       │  │
              ┌──────┐ ┌─┴────┐ ┌──────┐ ┌─┴────┴──┐   │       │  │
              │UART1 │ │UART2 │ │ PRN  │ │ DS12885 │   │       │  │
              │16450 │ │16450 │ │82C11 │ │  RTC    │   │       │  │
              │      │ │      │ │      │ │DS=IOR#  │   │       │  │
              │      │ │      │ │      │ │R/W=IOW# │   │       │  │
              └──┬───┘ └──┬───┘ └──┬───┘ └────┬────┘   │       │  │
                 │IRQ     │IRQ    │IRQ        │IRQ     │       │  │
                 │        │       │           │        │       │  │
              [JP_IRQ] [JP_IRQ] [JP_IRQ]  [JP_IRQ]    │       │  │
                 └────────┴───────┴───────────┴────────┘       │  │
                                                    To ISA IRQ lines
```

## 11. Implementation Steps

1. **Create KiCad schematic** with all components
2. **Program ATF22V10** with CUPL equations using a programmer (e.g., TL866II+)
3. **Verify address decode** with logic analyzer before populating all ICs
4. **PCB layout** following standard ISA card dimensions (XT 8-bit: 4.8" x 3.7" approx)
5. **Test each subsystem** independently (RTC first, then UARTs, then PRN)

## 12. Design Notes and Caveats

- The ATF22V10C-15PU (15ns) is fast enough for ISA bus timing (ISA bus cycle ~500ns minimum).
- All address jumper pull-down resistors (10k) ensure a defined state when no jumper cap is installed (defaults to Option B / lower address).
- All enable jumper pull-up resistors (10k) ensure devices are disabled when no jumper cap is installed (safe default).
- The DS12885 IRQ output is open-drain — it MUST have an external pull-up resistor (4.7k to VCC).
- UART IRQ outputs may need to be open-collector/drain if sharing IRQ lines. Check the specific UART chip datasheet.
- The 74HCT245 variant (not 74HC245) is recommended for proper TTL-level compatibility with the ISA bus.
- CR2032 battery: connect positive terminal to DS12885 VBAT, negative to GND. Add a series diode (1N4148) or Schottky diode to prevent reverse charging (optional, DS12885 is UL recognized for this).
