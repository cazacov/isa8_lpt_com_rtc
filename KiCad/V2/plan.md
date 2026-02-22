# ISA8 LPT COM RTC - Implementation Plan

## 1. ISA Bus Signals Used

| ISA Signal | Direction | Purpose |
|---|---|---|
| D0-D7 | Bidir | Data bus (active via 74245) |
| A0-A9 | In | Address bus (only A0-A9 needed, highest port 0x3F8 = 0b11_1111_1000; A2 also used for precise RTC decode) |
| AEN | In | DMA cycle indicator. Must be LOW for valid CPU I/O cycle |
| IOR# | In | I/O Read strobe (active low) |
| IOW# | In | I/O Write strobe (active low) |
| IRQ3-IRQ7 | Out | Interrupt request lines (active high) |

## 2. I/O Port Address Map

| Device | Option A (Default) | Option B | Address Range Size |
|---|---|---|---|
| COM1 | 0x3F8-0x3FF | 0x2F8-0x2FF | 8 bytes |
| COM2 | 0x3E8-0x3EF | 0x2E8-0x2EF | 8 bytes |
| PRN (LPT1) | 0x378-0x37F | 0x278-0x27F (LPT2) | 8 bytes |
| RTC | 0x70-0x71 | (fixed) | 2 bytes |

### Address Bit Patterns (A9..A0)

```
COM1 @ 0x3F8:  11 1111 1xxx    COM1 @ 0x2F8:  10 1111 1xxx
COM2 @ 0x3E8:  11 1110 1xxx    COM2 @ 0x2E8:  10 1110 1xxx
PRN   @ 0x378:  11 0111 1xxx    PRN   @ 0x278:  10 0111 1xxx
RTC   @ 0x070:  00 0111 00 0x   (fixed, no jumper; A2 checked by PLD)
```

Note: `x` bits are don't-care for chip-select (decoded internally by each device via A0-A2). For the RTC, A0 and A2 are checked by the PLD; A1 is not checked (mirroring at 0x72/0x73 only — unused addresses on any standard PC).

## 3. Jumper Design

### 3.1 Jumper Inputs to ATF22V10

Instead of feeding raw address lines and muxing them inside the PLD, use jumpers to route one of two address lines to a single PLD input. This minimizes PLD pin usage.

The key observation: for each device pair, only **one address bit** changes between Option A and Option B — and that bit is always **A8**:

| Device | Option A | Option B | Differing Bit |
|---|---|---|---|
| COM1 | 0x3F8 (A8=1) | 0x2F8 (A8=0) | A8 |
| COM2 | 0x3E8 (A8=1) | 0x2E8 (A8=0) | A8 |
| PRN | 0x378 (A8=1) | 0x278 (A8=0) | A8 |

All three devices have A9=1 for both address options. Each device needs its own jumper input to the PLD for independent address configuration.

**Approach: one jumper bit per device fed into the PLD.**

- **J_COM1**: 1-bit jumper (HIGH = 0x3F8, LOW = 0x2F8)
- **J_COM2**: 1-bit jumper (HIGH = 0x3E8, LOW = 0x2E8)
- **J_PRN**: 1-bit jumper (HIGH = 0x378, LOW = 0x278)

These jumper signals connect directly to ATF22V10 inputs. The PLD equations incorporate them.

### 3.2 IRQ Selection Jumpers

IRQ routing does NOT go through the PLD. Use simple 3-pin jumper headers connecting each device's IRQ output to the selected ISA IRQ line.

| Jumper | Position 1 | Position 2 | Notes |
|---|---|---|---|
| J_IRQ_COM1 | IRQ4 | IRQ3 | Standard COM1=IRQ4, COM2=IRQ3 |
| J_IRQ_COM2 | IRQ4 | IRQ3 | Standard COM3=IRQ4, COM4=IRQ3 |
| J_IRQ_PRN | IRQ7 | IRQ5 | Standard LPT1=IRQ7, LPT2=IRQ5 |
| J_IRQ_RTC | IRQ7 / IRQ6 / IRQ5 | — | 4-pin header, jumper selects one of 3 |

For the RTC IRQ (3 choices: IRQ5, IRQ6, IRQ7), use a 1x4 pin header:
```
Pin 1: RTC_IRQ (from DS12885 IRQ pin, active low with external pull-up)
Pin 2: IRQ5
Pin 3: IRQ6
Pin 4: IRQ7
```
A single jumper wire connects Pin 1 to one of Pins 2-4.

### 3.3 Jumper Summary Table

| Jumper | Type | Pins | Function |
|---|---|---|---|
| JP_COM1_ADDR | 3-pin header | 1=VCC, 2=J_COM1, 3=GND | Cap 1-2: 0x3F8 (COM1), Cap 2-3: 0x2F8 (COM2) |
| JP_COM1_IRQ | 3-pin header | 1=IRQ4, 2=COM1_IRQ, 3=IRQ3 | Select IRQ |
| JP_COM2_ADDR | 3-pin header | 1=VCC, 2=J_COM2, 3=GND | Cap 1-2: 0x3E8 (COM3), Cap 2-3: 0x2E8 (COM4) |
| JP_COM2_IRQ | 3-pin header | 1=IRQ4, 2=COM2_IRQ, 3=IRQ3 | Select IRQ |
| JP_PRN_ADDR | 3-pin header | 1=VCC, 2=J_PRN, 3=GND | Cap 1-2: 0x378 (LPT1), Cap 2-3: 0x278 (LPT2) |
| JP_PRN_IRQ | 3-pin header | 1=IRQ7, 2=PRN_IRQ, 3=IRQ5 | Select IRQ |
| JP_RTC_IRQ | 1x4 header | 1=RTC_IRQ, 2=IRQ5, 3=IRQ6, 4=IRQ7 | Jumper wire Pin1 to one of Pin2-4 |

### 3.4 Device Enable/Disable

Devices are enabled/disabled by installing or removing the chip from its DIP socket. A 10k×8 pull-up resistor network (RN1) on the 74HCT245 B-side ensures empty sockets read as 0xFF. Software detection (e.g., BIOS UART scratch register test) correctly identifies absent devices. No PLD input pins or jumpers are needed for device enable/disable.

**Address jumper wiring (3-pin header):**

All three address jumpers (JP_COM1_ADDR, JP_COM2_ADDR, JP_PRN_ADDR) follow the same scheme:

```
  Pin 1 = VCC
  Pin 2 = J_xxx (to PLD input, 10k pull-down to GND)
  Pin 3 = GND
```
- Cap on 1-2: signal = HIGH → Option A (higher address)
- Cap on 2-3: signal = LOW → Option B (lower address)
- No cap: signal = LOW (pull-down) → Option B (default)

The PLD equation uses XNOR(A8, J_xxx) to check whether the bus address bit A8 matches the jumper setting.

## 4. ATF22V10 Pin Assignment

### 4.1 ATF22V10C Pinout (24-pin DIP/PLCC)

- Pins 1, 2-11: Inputs (pin 1 is clock/input, pins 2-11 are dedicated inputs)
- Pins 14-23: I/O (can be output or feedback)
- Pin 12: GND
- Pin 24: VCC

### 4.2 Pin Assignment

| ATF22V10 Pin | Direction | Signal | Purpose |
|---|---|---|---|
| 1 | Input | A0 | Address bit 0 (for RTC decode: port 0x70 vs 0x71) |
| 2 | Input | A2 | Address bit 2 (for precise RTC decode, eliminates 0x74-0x77 mirroring) |
| 3 | Input | A3 | Address bit 3 |
| 4 | Input | A4 | Address bit 4 |
| 5 | Input | A5 | Address bit 5 |
| 6 | Input | A6 | Address bit 6 |
| 7 | Input | A7 | Address bit 7 |
| 8 | Input | A8 | Address bit 8 |
| 9 | Input | A9 | Address bit 9 |
| 10 | Input | IOR# | ISA I/O Read strobe (active low), used for RTC_DS# generation |
| 11 | Input | IOW# | ISA I/O Write strobe (active low), used for RTC_AS and RTC_WR# generation |
| 12 | — | GND | Ground |
| 13 | Input | AEN | DMA indicator (active high = DMA, must be 0 for I/O) |
| 14 | Output | COM1_CS# | COM1 chip select (active low) |
| 15 | Output | COM2_CS# | COM2 chip select (active low) |
| 16 | Output | PRN_CS# | PRN chip select (active low) |
| 17 | Output | RTC_WR# | RTC write strobe (active low) → DS12885 WR# |
| 18 | Output | BUF_EN# | 74245 enable (active low) |
| 19 | Output | RTC_AS | DS12885 Address Strobe (active high, gated with IOW# + address decode) |
| 20 | Output | RTC_DS# | RTC data strobe (active low) → DS12885 DS# |
| 21 | Input | J_COM1 | Jumper: COM1 address select (1=0x3F8, 0=0x2F8) |
| 22 | Input | J_COM2 | Jumper: COM2 address select (1=0x3E8, 0=0x2E8) |
| 23 | Input | J_PRN | Jumper: PRN address select (1=0x378, 0=0x278) |
| 24 | — | VCC | +5V |

**Note:** A1 is not needed for chip-select decode — it is decoded internally by each peripheral (UART uses A0-A2, PRN uses A0-A2). A0 and A2 are on dedicated input pins 1–2 for compact address bus routing. A2 is checked by the PLD for the RTC address decode to prevent mirroring at 0x74-0x77 (the slave 8259A range on AT-class machines); mirroring at 0x72-0x73 (A1 unchecked) is harmless since those addresses are unused on any standard PC. IOR# and IOW# are on dedicated input pins 10–11 for RTC strobe generation. AEN is on I/O pin 13 (configured as input). All three address jumpers (J_COM1, J_COM2, J_PRN) are grouped on adjacent I/O pins 21–23 for a clean jumper layout. All 24 pins are assigned — no spare pins remain. Devices are enabled/disabled by installing or removing chips from sockets.

## 5. DS12885 RTC Interface to ISA Bus

### 5.1 Bus Mode Selection

Intel bus timing mode is selected by leaving MOT pin unconnected (internal 20k pull-down to GND).

### 5.2 Signal Connections

| DS12885 Pin | Signal | ISA / Board Connection |
|---|---|---|
| MOT | Bus mode | Leave unconnected (pulled low internally → Intel mode) |
| AD0-AD7 | Mux Addr/Data | Connected to card-side data bus (74245 B-side). These are shared address AND data lines. |
| CS | Chip select | Tie to GND (always selected — access is controlled by PLD-qualified strobes) |
| AS | Address strobe | RTC_AS from ATF22V10 — active-high, gated with IOW# and address decode (see below) |
| DS# | Data strobe | RTC_DS# from ATF22V10 — active-low, fires only during IN from port 0x71 |
| WR# | Write strobe | RTC_WR# from ATF22V10 — active-low, fires only during OUT to port 0x71 |
| RESET | Reset | Tie to VCC (per datasheet: allows power-fail transitions without clearing control registers) |
| IRQ | Interrupt out | Open-drain, connected via jumper to IRQ5/6/7, needs 4.7k pull-up to VCC |
| SQW | Square wave | Leave unconnected or route to test point |
| X1, X2 | Crystal | 32.768 kHz crystal (Y1) |
| VCC | Power | +5V from ISA bus |
| VBAT | Backup battery | CR2032 (3V) via battery holder |
| RCLR | RAM clear | Tie to VCC via 10k pull-up (or add a push-button to GND for manual RAM clear) |

### 5.3 Multiplexed Address/Data Bus — Address Latch Mechanism

The DS12885 uses a **multiplexed** address/data bus (AD0-AD7). The CPU first writes a register address to port 0x70, then reads/writes data at port 0x71:

- **Port 0x70 (A0=0):** AS pulses HIGH → DS12885 latches AD0–AD7 as the internal register address.
- **Port 0x71 (A0=1):** AS stays LOW → normal data read/write to the previously addressed register.

**Implementation:** `RTC_AS = !AEN & !IOW & !A0 & address_decode` — the AS signal is fully qualified with the IOW# strobe and RTC address decode inside the PLD. Per the DS12885 datasheet, address latching via AS occurs regardless of CS state (*"Bus cycles that take place without asserting CS will latch addresses, but no access occurs"*). This means a naive `AS = !A0` would corrupt the address latch during every non-RTC bus cycle (instruction fetches, other I/O), because AS would toggle and re-latch whatever residual charge remains on the floating AD0-AD7 lines. By gating AS with IOW# and address decode, the latch opens ONLY during the IOW# pulse of an `OUT 70h` instruction, guaranteeing valid data capture and preventing corruption between bus cycles.

### 5.4 Intel Mode Timing Summary

For a **write to port 0x70** (set RTC register address):
1. CPU puts 0x70 on address bus, AEN=0
2. ATF22V10 asserts BUF_EN# low. RTC_AS stays LOW (IOW# not yet active). RTC_WR# stays HIGH. DS12885 CS# is permanently GND.
3. CPU drives data (register address 0x00-0x7F) on D0-D7 → through 74245 → AD0-AD7
4. IOW# goes LOW → ATF22V10 asserts RTC_AS HIGH (address match + A0=0 + IOW# active) → DS12885 AS goes HIGH → address latch becomes **transparent** → AD0-AD7 value (register address) flows through to the internal register address. RTC_WR# stays HIGH (A0=0, not a data write).
5. IOW# returns HIGH → ATF22V10 deasserts RTC_AS → AS **falls** → address latch **freezes**, capturing the register address that was present on AD0-AD7
6. Bus cycle ends. Between bus cycles, RTC_AS stays LOW — the address latch is frozen and cannot be corrupted by intervening instruction fetches or other bus activity

For a **read from port 0x71** (read RTC register data):
1. CPU puts 0x71 on address bus, AEN=0
2. ATF22V10 asserts BUF_EN# low. AS stays LOW (A0=1) — address latch **stays frozen** with the previously set register address
3. IOR# goes LOW → ATF22V10 asserts RTC_DS# LOW (address match + A0=1 + IOR# active) → DS12885 DS# goes LOW → DS12885 drives register data onto AD0-AD7
4. 74245 direction = B→A (IOR# = 0 → DIR = 0) → data flows to ISA D0-D7
5. CPU reads data; IOR# goes HIGH → RTC_DS# goes HIGH → DS12885 releases bus

For a **write to port 0x71** (write RTC register data):
1. Same addressing as read — AS=0, address latch frozen
2. CPU drives write data on D0-D7 → through 74245 → AD0-AD7
3. IOW# goes LOW → ATF22V10 asserts RTC_WR# LOW (address match + A0=1 + IOW# active) → DS12885 WR# goes LOW
4. IOW# goes HIGH → RTC_WR# goes HIGH → WR# rising edge → DS12885 latches AD0-AD7 as write data

### 5.5 Address Latch Integrity Between Bus Cycles

Since RTC_AS is gated with IOW# and the full RTC address decode, the DS12885 address latch opens **only** during the IOW# pulse of an `OUT 70h` instruction. Between bus cycles, RTC_AS stays LOW and the address latch remains frozen — no corruption is possible.

This is critical because the DS12885 datasheet states that AS-triggered address latching occurs regardless of CS state. A simpler approach like `AS = !A0` would toggle the latch during every bus cycle, re-latching whatever residual charge remains on the floating AD0-AD7 lines — an unreliable scheme that depends on bus capacitance retention across multiple instruction fetch cycles.

## 6. 74245 Bus Buffer

### 6.1 Connections

| 74245 Pin | Signal | Connection |
|---|---|---|
| A1-A8 (pins 2-9) | ISA D0-D7 | ISA bus data lines |
| B1-B8 (pins 11-18) | Card D0-D7 | COM1, COM2, PRN data pins, DS12885 AD0-AD7 |
| DIR (pin 1) | Direction | Connected to IOR# from ISA bus |
| G# (pin 19) | Enable | BUF_EN# from ATF22V10 |

### 6.2 Direction Logic

- IOR# = 0 (I/O read active): DIR = 0 → B-to-A (device → ISA bus)
- IOR# = 1 (idle or I/O write): DIR = 1 → A-to-B (ISA bus → device)

### 6.3 Enable Logic

G# (BUF_EN#) is active low. The 74245 is enabled only when a device on this card is being addressed, preventing bus conflicts with other ISA cards.

## 7. Bill of Materials (Logic Section)

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
| RN1 | 10k×8 SIP resistor network | SIP-9 | 1 | Pull-up for 74HCT245 B-side (active device side) |
| R4 | 4.7k resistor | Axial | 1 | Pull-up for DS12885 IRQ (open-drain) |
| R5-R7 | 10k resistor | Axial | 3 | Pull-down for J_xxx addr jumpers |
| JP1-JP7 | Pin headers | 2.54mm | Various | Jumper configuration headers |

## 8. Schematic Block Diagram

```
                    ISA 8-bit Bus
                    ═══════════════════════════════════
                    │D0-D7  A0-A9  IOR# IOW# AEN  IRQ3-7│
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
    │ J_COM1/2 ◄──┼──┼─┼── Jumpers        │    │      │       │  │
    │ J_PRN     ◄──┼──┼─┼── Jumpers        │    │      │       │  │
    │              │  │ │                   │    │      │       │  │
    │ COM1_CS# ───┼──┼─┼──► COM1 CS2     │    │      │       │  │
    │ COM2_CS# ───┼──┼─┼──► COM2 CS2     │    │      │       │  │
    │ PRN_CS#   ───┼──┼─┼──► PRN CS        │    │      │       │  │
    │ RTC_WR#  ───┼──┼─┼──► DS12885 WR#  │    │      │       │  │
    │ BUF_EN#   ───┼──┘ │                  │    │      │       │  │
    │ RTC_AS   ───┼────►│  DS12885 AS     │    │      │       │  │
    │ RTC_DS#  ───┼────►│  DS12885 DS#    │    │      │       │  │
    │ IOR# ◄──────┼─────┼──────────────────┘    │      │       │  │
    └──────────────┘     │                  │    │      │       │  │
                         │                  │    │      │       │  │
                Card Data Bus (from 74245 B-side)│      │       │  │
                   ══════╪══════════════════╪════╪══════╪═══════╪══╪
                         │                  │    │      │       │  │
              ┌──────┐ ┌─┴────┐ ┌──────┐ ┌─┴────┴──┐   │       │  │
              │COM1  │ │COM2  │ │ PRN  │ │ DS12885 │   │       │  │
              │16450 │ │16450 │ │82C11 │ │  RTC    │   │       │  │
              │      │ │      │ │      │ │CS#=GND  │   │       │  │
              │      │ │      │ │      │ │WR#=PLD │   │       │  │
              │      │ │      │ │      │ │RD#=PLD │   │       │  │
              └──┬───┘ └──┬───┘ └──┬───┘ └────┬────┘   │       │  │
                 │IRQ     │IRQ    │IRQ        │IRQ     │       │  │
                 │        │       │           │        │       │  │
              [JP_IRQ] [JP_IRQ] [JP_IRQ]  [JP_IRQ]    │       │  │
                 └────────┴───────┴───────────┴────────┘       │  │
                                                    To ISA IRQ lines
```

## 9. Implementation Steps

1. **Create KiCad schematic** with all components
2. **Program ATF22V10** with CUPL equations using a programmer (e.g., TL866II+)
3. **Verify address decode** with logic analyzer before populating all ICs
4. **PCB layout** following standard ISA card dimensions (XT 8-bit: 4.8" x 3.7" approx)
5. **Test each subsystem** independently (RTC first, then UARTs, then PRN)

## 10. Design Notes and Caveats

- The ATF22V10C-15PU (15ns) is fast enough for ISA bus timing (ISA bus cycle ~500ns minimum).
- All address jumper pull-down resistors (10k) ensure a defined state when no jumper cap is installed (defaults to Option B / lower address).
- Devices are enabled/disabled by installing or removing chips from their sockets. A 10k×8 SIP pull-up network on the 74HCT245 B-side keeps all data lines at defined HIGH levels when a socket is empty, preventing bus contention.
- The DS12885 IRQ output is open-drain — it MUST have an external pull-up resistor (4.7k to VCC).
- UART IRQ outputs may need to be open-collector/drain if sharing IRQ lines. Check the specific UART chip datasheet.
- The 74HCT245 variant (not 74HC245) is recommended for proper TTL-level compatibility with the ISA bus.
- CR2032 battery: connect positive terminal to DS12885 VBAT, negative to GND. Add a series diode (1N4148) or Schottky diode to prevent reverse charging (optional, DS12885 is UL recognized for this).

## 11. ATF22V10 Chip Logic — Detailed Analysis

### 11.1 XNOR Address Matching

For each configurable device (COM1, COM2, PRN), the two alternative base addresses differ only in A8. The PLD uses XNOR(A8, J_xxx) to check whether the bus address matches the jumper setting:

$$\text{A8\_MATCH} = \text{A8} \odot \text{J\_xxx} = (\text{A8} \cdot \text{J\_xxx}) + (\overline{\text{A8}} \cdot \overline{\text{J\_xxx}})$$

Since the ATF22V10 cannot express XNOR as a single gate, each chip-select equation expands into two product terms (one for A8=1∧J=1, one for A8=0∧J=0). The remaining address bits (A9, A7–A3) are checked against fixed expected values. Every equation also requires AEN=0 (not a DMA cycle).

### 11.2 PLD Equations

**COM1** (0x3F8 or 0x2F8 — fixed bits: A9=1, A7–A3=11111):

```
/COM1 = /AEN * A9 *  A8 *  JCOM1 * A7 * A6 * A5 * A4 * A3
       + /AEN * A9 * /A8 * /JCOM1 * A7 * A6 * A5 * A4 * A3
```

**COM2** (0x3E8 or 0x2E8 — same as COM1 but A4=0):

```
/COM2 = /AEN * A9 *  A8 *  JCOM2 * A7 * A6 * A5 * /A4 * A3
       + /AEN * A9 * /A8 * /JCOM2 * A7 * A6 * A5 * /A4 * A3
```

**PRN** (0x378 or 0x278 — same structure but A7=0):

```
/PRN = /AEN * A9 *  A8 *  JPRN * /A7 * A6 * A5 * A4 * A3
     + /AEN * A9 * /A8 * /JPRN * /A7 * A6 * A5 * A4 * A3
```

**RTC Write Strobe** (port 0x71 write — DS12885 WR#, 1 product term):

```
/RTC_WR = /AEN * /IOW * A0 * /A2 * /A9 * /A8 * /A7 * A6 * A5 * A4 * /A3
```

**BUFEN** — logical OR of all chip-select conditions. The CUPL source references the output signal names (`COM1_CS # COM2_CS # ...`), and the compiler expands them inline into the full product-term equations (7 terms total ≤ 16 available). This avoids pin-feedback delay and ensures BUF_EN# asserts simultaneously with the chip-select outputs, both with a single PLD propagation delay from input change:

```
/BUFEN = /AEN * A9 *  A8 *  JCOM1 * A7 * A6 * A5 * A4 * A3
       + /AEN * A9 * /A8 * /JCOM1 * A7 * A6 * A5 * A4 * A3
       + /AEN * A9 *  A8 *  JCOM2 * A7 * A6 * A5 * /A4 * A3
       + /AEN * A9 * /A8 * /JCOM2 * A7 * A6 * A5 * /A4 * A3
       + /AEN * A9 *  A8 *  JPRN   * /A7 * A6 * A5 * A4 * A3
       + /AEN * A9 * /A8 * /JPRN   * /A7 * A6 * A5 * A4 * A3
       + /AEN * /A2 * /A9 * /A8 * /A7 * A6 * A5 * A4 * /A3
```

**RTC_AS** — address strobe for DS12885, gated with IOW# and full RTC address decode. Goes HIGH only during a write to port 0x70 (IOW# active, A0=0, address matches). This ensures the address latch opens exclusively during the IOW# pulse of `OUT 70h`, preventing corruption between bus cycles:

```
RTC_AS = /AEN * /IOW * /A0 * /A2 * /A9 * /A8 * /A7 * A6 * A5 * A4 * /A3
```

**RTC Data Strobe** (port 0x71 read — DS12885 DS#, 1 product term):

```
/RTC_DS = /AEN * /IOR * A0 * /A2 * /A9 * /A8 * /A7 * A6 * A5 * A4 * /A3
```

### 11.3 Product Term Budget

| Output Pin | Signal | Available PTs | Used PTs | Utilization |
|------------|--------|---------------|----------|-------------|
| 14 | /COM1 | 8 | 2 | 25% |
| 15 | /COM2 | 10 | 2 | 20% |
| 16 | /PRN | 12 | 2 | 17% |
| 17 | /RTC_WR | 16 | 1 | 6% |
| 18 | /BUFEN | 16 | 7 | 44% |
| 19 | RTC_AS | 14 | 1 | 7% |
| 20 | /RTC_DS | 12 | 1 | 8% |
| 21 | input (J_COM1) | — | 0 | — |
| 22 | input (J_COM2) | — | 0 | — |
| 23 | input (J_PRN) | — | 0 | — |

Total: **16 product terms** used, substantial headroom on all pins.

### 11.4 CUPL Source

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
Pin 1  = A0;    /* ISA address bit 0 (RTC port 0x70 vs 0x71) */
Pin 2  = A2;    /* ISA address bit 2 (precise RTC decode) */
Pin 3  = A3;
Pin 4  = A4;
Pin 5  = A5;
Pin 6  = A6;
Pin 7  = A7;
Pin 8  = A8;
Pin 9  = A9;
Pin 10 = IOR;   /* ISA IOR# (active low) */
Pin 11 = IOW;   /* ISA IOW# (active low) */
Pin 13 = AEN;   /* ISA AEN (DMA indicator) */

/* Address jumpers on I/O pins (configured as inputs) */
Pin 21 = J_COM1;
Pin 22 = J_COM2;

Pin 23 = J_PRN;

/* Outputs (active-low) */
Pin 14 = !COM1_CS;
Pin 15 = !COM2_CS;
Pin 16 = !PRN_CS;
Pin 17 = !RTC_WR;
Pin 18 = !BUF_EN;
Pin 19 = RTC_AS;
Pin 20 = !RTC_DS;

/* COM1: 0x3F8 or 0x2F8 — differ in A8 */
/* Common: A9=1, A7=1, A6=1, A5=1, A4=1, A3=1 */
COM1_CS = !AEN
         & A9 & (A8 $ !J_COM1) & A7 & A6 & A5 & A4 & A3;

/* COM2: 0x3E8 or 0x2E8 — differ in A8 */
/* Common: A9=1, A7=1, A6=1, A5=1, A4=0, A3=1 */
COM2_CS = !AEN
         & A9 & (A8 $ !J_COM2) & A7 & A6 & A5 & !A4 & A3;

/* PRN: 0x378 or 0x278 — differ in A8 */
/* Common: A9=1, A7=0, A6=1, A5=1, A4=1, A3=1 */
PRN_CS = !AEN
       & A9 & (A8 $ !J_PRN) & !A7 & A6 & A5 & A4 & A3;

/* RTC Write Strobe: drives DS12885 WR# (Intel mode) */
/* Fires during OUT to port 0x71 (A0=1, data register access) */
/* DS12885 CS# is tied to GND — access controlled by PLD-qualified strobes */
RTC_WR = !AEN & !IOW & A0 & !A2 & !A9 & !A8 & !A7 & A6 & A5 & A4 & !A3;

/* Buffer enable: active when any device address range is on the bus */
BUF_EN = COM1_CS # COM2_CS # PRN_CS
       # (!AEN & !A2 & !A9 & !A8 & !A7 & A6 & A5 & A4 & !A3);

/* RTC Address Strobe: drives DS12885 AS (Intel mode) */
/* Goes HIGH only during OUT 70h (IOW# active, A0=0, RTC address match) */
/* Prevents address latch corruption between bus cycles */
RTC_AS = !AEN & !IOW & !A0 & !A2 & !A9 & !A8 & !A7 & A6 & A5 & A4 & !A3;

/* RTC Data Strobe: drives DS12885 DS# (Intel mode) */
/* Fires during IN from port 0x71 (A0=1, data register access) */
RTC_DS = !AEN & !IOR & A0 & !A2 & !A9 & !A8 & !A7 & A6 & A5 & A4 & !A3;
```
