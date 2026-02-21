# ISA8 LPT COM RTC - Implementation Plan

## 1. ISA Bus Signals Used

| ISA Signal | Direction | Purpose |
|---|---|---|
| D0-D7 | Bidir | Data bus (active via 74245) |
| A0-A9 | In | Address bus (only A0-A9 needed, highest port 0x3F8 = 0b11_1111_1000) |
| AEN | In | DMA cycle indicator. Must be LOW for valid CPU I/O cycle |
| IOR# | In | I/O Read strobe (active low) |
| IOW# | In | I/O Write strobe (active low) |
| IRQ3-IRQ7 | Out | Interrupt request lines (active high) |
| RESET DRV | In | System reset |

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
RTC   @ 0x070:  00 0111 000x    (fixed, no jumper)
```

Note: `x` bits are don't-care for chip-select (decoded internally by each device via A0-A2).

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

### 3.2 Device Enable/Disable Jumpers

To disable a device completely (nice-to-have from spec):

- **EN_COM1**: jumper, when open (pulled HIGH via resistor) = disabled. When shorted to GND = enabled.
- **EN_COM2**: same scheme.
- **EN_PRN**: same scheme.

These also go into the ATF22V10 as inputs. When the enable input is HIGH (disabled), the PLD simply never asserts the corresponding CS output.

### 3.3 IRQ Selection Jumpers

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

### 3.4 Jumper Summary Table

| Jumper | Type | Pins | Function |
|---|---|---|---|
| JP_COM1_ADDR | 3-pin header | 1=VCC, 2=J_COM1, 3=GND | Cap 1-2: 0x3F8 (COM1), Cap 2-3: 0x2F8 (COM2) |
| JP_COM1_EN | 2-pin header | 1=COM1_EN, 2=GND | Shorted = enabled, open = disabled (pull-up) |
| JP_COM1_IRQ | 3-pin header | 1=IRQ4, 2=COM1_IRQ, 3=IRQ3 | Select IRQ |
| JP_COM2_ADDR | 3-pin header | 1=VCC, 2=J_COM2, 3=GND | Cap 1-2: 0x3E8 (COM3), Cap 2-3: 0x2E8 (COM4) |
| JP_COM2_EN | 2-pin header | 1=COM2_EN, 2=GND | Shorted = enabled, open = disabled (pull-up) |
| JP_COM2_IRQ | 3-pin header | 1=IRQ4, 2=COM2_IRQ, 3=IRQ3 | Select IRQ |
| JP_PRN_ADDR | 3-pin header | 1=VCC, 2=J_PRN, 3=GND | Cap 1-2: 0x378 (LPT1), Cap 2-3: 0x278 (LPT2) |
| JP_PRN_EN | 2-pin header | 1=PRN_EN, 2=GND | Shorted = enabled, open = disabled (pull-up) |
| JP_PRN_IRQ | 3-pin header | 1=IRQ7, 2=PRN_IRQ, 3=IRQ5 | Select IRQ |
| JP_RTC_IRQ | 1x4 header | 1=RTC_IRQ, 2=IRQ5, 3=IRQ6, 4=IRQ7 | Jumper wire Pin1 to one of Pin2-4 |

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
| 10 | Input | J_COM1 | Jumper: COM1 address select (1=0x3F8, 0=0x2F8) |
| 11 | Input | J_COM2 | Jumper: COM2 address select (1=0x3E8, 0=0x2E8) |
| 12 | — | GND | Ground |
| 13 | Input | J_PRN | Jumper: PRN address select (1=0x378, 0=0x278) |
| 14 | Output | COM1_CS# | COM1 chip select (active low) |
| 15 | Output | COM2_CS# | COM2 chip select (active low) |
| 16 | Output | PRN_CS# | PRN chip select (active low) |
| 17 | Output | RTC_CS# | RTC chip select (active low) |
| 18 | Output | BUF_EN# | 74245 enable (active low) |
| 19 | Output | RTC_AS | DS12885 Address Strobe (active high = !A0) |
| 20 | Input | EN_PRN# | PRN enable jumper (0=enabled, active when shorted to GND) |
| 21 | Input | EN_COM2# | COM2 enable jumper (0=enabled) |
| 22 | Input | EN_COM1# | COM1 enable jumper (0=enabled) |
| 23 | Output | (spare) | |
| 24 | — | VCC | +5V |

**Note:** A1 and A2 are not needed for chip-select decode — they are decoded internally by each peripheral (UART uses A0-A2, PRN uses A0-A2). We only need A0 for the RTC (which uses port 0x70 and 0x71, differing by A0).

## 5. DS12885 RTC Interface to ISA Bus

### 5.1 Bus Mode Selection

Intel bus timing mode is selected by leaving MOT pin unconnected (internal 20k pull-down to GND).

### 5.2 Signal Connections

| DS12885 Pin | Signal | ISA / Board Connection |
|---|---|---|
| MOT | Bus mode | Leave unconnected (pulled low internally → Intel mode) |
| AD0-AD7 | Mux Addr/Data | Connected to card-side data bus (74245 B-side). These are shared address AND data lines. |
| CS | Chip select | RTC_CS# from ATF22V10 (active low) |
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

The DS12885 uses a **multiplexed** address/data bus (AD0-AD7). The CPU first writes a register address to port 0x70, then reads/writes data at port 0x71:

- **Port 0x70 (A0=0):** AS pulses HIGH → DS12885 latches AD0–AD7 as the internal register address.
- **Port 0x71 (A0=1):** AS stays LOW → normal data read/write to the previously addressed register.

**Implementation:** `RTC_AS = !A0` — a simple inverter implemented inside the PLD. Per the DS12885 datasheet, address latching via AS occurs regardless of CS state (*"Bus cycles that take place without asserting CS will latch addresses, but no access occurs"*). Since no data read or write can occur without CS asserted, spurious AS edges from non-RTC bus activity are harmless — they may re-latch the address, but never cause an unintended data access. Therefore no address-matching terms are needed in the RTC_AS equation.

### 5.4 Intel Mode Timing Summary

For a **write to port 0x70** (set RTC register address):
1. CPU puts 0x70 on address bus, AEN=0
2. ATF22V10 asserts RTC_CS# low, BUF_EN# low
3. AS = !A0 = 1 (A0=0 for port 0x70) → the DS12885's internal address latch is **transparent** — AD0–AD7 flows through to the internal register address
4. CPU drives data (register address 0x00-0x7F) on D0-D7 → through 74245 → AD0-AD7 → flows through the transparent latch as the register address
5. IOW# (R/W) goes LOW then HIGH — but the DS12885 is still in the address phase (AS=1). The device does **not** perform a data write because the data-phase timing (tASED ≥ 40 ns after AS fall) has not been met. The R/W pulse is effectively ignored.
6. Bus cycle ends → A0 changes → AS = !A0 **falls** → the address latch **freezes**, capturing the register address that was present on AD0–AD7 while AS was HIGH

For a **read from port 0x71** (read RTC register data):
1. CPU puts 0x71 on address bus, AEN=0
2. ATF22V10 asserts RTC_CS# low, BUF_EN# low
3. AS = !A0 = 0 (A0=1 for port 0x71) — address latch **stays frozen** with the previously set register address
4. IOR# goes low → DS pin (= IOR# in Intel mode, acts as OE) goes low → DS12885 drives register data onto AD0-AD7
5. 74245 direction = B→A (IOR# = 0 → DIR = 0) → data flows to ISA D0-D7
6. CPU reads data; IOR# goes HIGH → DS12885 releases bus

For a **write to port 0x71** (write RTC register data):
1. Same addressing as read — AS=0, address latch frozen, CS asserted
2. CPU drives write data on D0-D7 → through 74245 → AD0-AD7
3. IOW# (R/W) goes LOW then HIGH → DS12885 latches AD0-AD7 as write data on R/W rising edge

### 5.5 Address Latch Integrity Between Bus Cycles

Since `RTC_AS = !A0` toggles with every bus cycle (instruction fetches, memory reads, etc.), the DS12885's address latch opens and closes continuously between the OUT 70h and IN/OUT 71h instructions. This raises the question: does the latched register address get corrupted?

**Why it works — bus capacitance retention:**

1. During OUT 70h, the 74245 drives AD0-AD7 with the register address value. When the bus cycle ends, 74245 is disabled (BUF_EN# goes HIGH).
2. The AD0-AD7 pins have ~7 pF parasitic capacitance (per DS12885 datasheet: CIO = 7 pF typical) and ≤ 1 µA leakage current. The 74245 B-side in high-Z adds similar characteristics.
3. Between OUT 70h and IN/OUT 71h, the 8088 executes 0–2 instruction fetches (~840 ns at 4.77 MHz). Voltage droop on AD0-AD7 during this time: $\Delta V = I \cdot t / C = 1\,\mu A \times 840\,ns / 7\,pF \approx 120\,mV$ — negligible vs. logic thresholds (VIH min = 2.2 V, VIL max = 0.8 V).
4. When AS toggles during instruction fetches, the latch opens (transparent) and closes (latches) with approximately the **same value** that was driven during OUT 70h, because bus capacitance preserves it.
5. When the IN/OUT 71h cycle starts, AS = 0 (latch frozen). The latched address is the value retained by bus capacitance — which matches the register address written by OUT 70h.

**Additional safety:** Even if the latched address were somehow corrupted, the only consequence would be reading/writing the wrong RTC register. No data access occurs during intervening bus cycles because CS is not asserted (RTC_CS# is HIGH when the address bus doesn't match 0x70-0x77).

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
| R1-R3 | 10k resistor | Axial | 3 | Pull-up for EN_xxx# jumpers |
| R4 | 4.7k resistor | Axial | 1 | Pull-up for DS12885 IRQ (open-drain) |
| R5-R7 | 10k resistor | Axial | 3 | Pull-down for J_xxx addr jumpers |
| JP1-JP10 | Pin headers | 2.54mm | Various | Jumper configuration headers |

## 8. Schematic Block Diagram

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
    │ J_COM1/2 ◄──┼──┼─┼── Jumpers        │    │      │       │  │
    │ J_PRN     ◄──┼──┼─┼── Jumpers        │    │      │       │  │
    │ EN_*#     ◄──┼──┼─┼── Jumpers        │    │      │       │  │
    │              │  │ │                   │    │      │       │  │
    │ COM1_CS# ───┼──┼─┼──► COM1 CS2     │    │      │       │  │
    │ COM2_CS# ───┼──┼─┼──► COM2 CS2     │    │      │       │  │
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
              │COM1  │ │COM2  │ │ PRN  │ │ DS12885 │   │       │  │
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

## 9. Implementation Steps

1. **Create KiCad schematic** with all components
2. **Program ATF22V10** with CUPL equations using a programmer (e.g., TL866II+)
3. **Verify address decode** with logic analyzer before populating all ICs
4. **PCB layout** following standard ISA card dimensions (XT 8-bit: 4.8" x 3.7" approx)
5. **Test each subsystem** independently (RTC first, then UARTs, then PRN)

## 10. Design Notes and Caveats

- The ATF22V10C-15PU (15ns) is fast enough for ISA bus timing (ISA bus cycle ~500ns minimum).
- All address jumper pull-down resistors (10k) ensure a defined state when no jumper cap is installed (defaults to Option B / lower address).
- All enable jumper pull-up resistors (10k) ensure devices are disabled when no jumper cap is installed (safe default).
- The DS12885 IRQ output is open-drain — it MUST have an external pull-up resistor (4.7k to VCC).
- UART IRQ outputs may need to be open-collector/drain if sharing IRQ lines. Check the specific UART chip datasheet.
- The 74HCT245 variant (not 74HC245) is recommended for proper TTL-level compatibility with the ISA bus.
- CR2032 battery: connect positive terminal to DS12885 VBAT, negative to GND. Add a series diode (1N4148) or Schottky diode to prevent reverse charging (optional, DS12885 is UL recognized for this).

## 11. ATF22V10 Chip Logic — Detailed Analysis

### 11.1 XNOR Address Matching

For each configurable device (COM1, COM2, PRN), the two alternative base addresses differ only in A8. The PLD uses XNOR(A8, J_xxx) to check whether the bus address matches the jumper setting:

$$\text{A8\_MATCH} = \text{A8} \odot \text{J\_xxx} = (\text{A8} \cdot \text{J\_xxx}) + (\overline{\text{A8}} \cdot \overline{\text{J\_xxx}})$$

Since the ATF22V10 cannot express XNOR as a single gate, each chip-select equation expands into two product terms (one for A8=1∧J=1, one for A8=0∧J=0). The remaining address bits (A9, A7–A3) are checked against fixed expected values. Every equation also requires AEN=0 (not a DMA cycle) and the device's enable jumper to be active.

### 11.2 PLD Equations

**COM1** (0x3F8 or 0x2F8 — fixed bits: A9=1, A7–A3=11111):

```
/COM1 = /AEN * /ECOM1 * A9 *  A8 *  JCOM1 * A7 * A6 * A5 * A4 * A3
       + /AEN * /ECOM1 * A9 * /A8 * /JCOM1 * A7 * A6 * A5 * A4 * A3
```

**COM2** (0x3E8 or 0x2E8 — same as COM1 but A4=0):

```
/COM2 = /AEN * /ECOM2 * A9 *  A8 *  JCOM2 * A7 * A6 * A5 * /A4 * A3
       + /AEN * /ECOM2 * A9 * /A8 * /JCOM2 * A7 * A6 * A5 * /A4 * A3
```

**PRN** (0x378 or 0x278 — same structure but A7=0):

```
/PRN = /AEN * /ENPRN * A9 *  A8 *  JPRN * /A7 * A6 * A5 * A4 * A3
     + /AEN * /ENPRN * A9 * /A8 * /JPRN * /A7 * A6 * A5 * A4 * A3
```

**RTC** (0x70–0x71 fixed, no jumper/enable — 1 product term):

```
/RTC = /AEN * /A9 * /A8 * /A7 * A6 * A5 * A4 * /A3
```

> **Note:** A1 and A2 are not decoded, so RTC_CS# responds to the full 8-byte range 0x70–0x77. Unlike the UARTs and PRN (which internally decode A0–A2 into 8 separate registers), the DS12885 has no address pins — it uses the multiplexed AD0–AD7 bus. Ports 0x72–0x77 act as functional aliases of 0x70/0x71 (even addresses → address phase, odd addresses → data phase). This matches the original IBM AT behavior, where the chipset also does not fully decode A1–A2 for the RTC.

**BUFEN** — logical OR of all chip-select conditions. The CUPL source references the output signal names (`COM1_CS # COM2_CS # ...`), and the compiler expands them inline into the full product-term equations (7 terms total ≤ 16 available). This avoids pin-feedback delay and ensures BUF_EN# asserts simultaneously with the chip-select outputs, both with a single PLD propagation delay from input change:

```
/BUFEN = /AEN * /ECOM1 * A9 *  A8 *  JCOM1 * A7 * A6 * A5 * A4 * A3
       + /AEN * /ECOM1 * A9 * /A8 * /JCOM1 * A7 * A6 * A5 * A4 * A3
       + /AEN * /ECOM2 * A9 *  A8 *  JCOM2 * A7 * A6 * A5 * /A4 * A3
       + /AEN * /ECOM2 * A9 * /A8 * /JCOM2 * A7 * A6 * A5 * /A4 * A3
       + /AEN * /ENPRN  * A9 *  A8 *  JPRN   * /A7 * A6 * A5 * A4 * A3
       + /AEN * /ENPRN  * A9 * /A8 * /JPRN   * /A7 * A6 * A5 * A4 * A3
       + /AEN * /A9 * /A8 * /A7 * A6 * A5 * A4 * /A3
```

**RTCAS** — simple inverter; DS12885 CS gates whether AS is acted upon:

```
RTCAS = /A0
```

### 11.3 Product Term Budget

| Output Pin | Signal | Available PTs | Used PTs | Utilization |
|------------|--------|---------------|----------|-------------|
| 14 | /COM1 | 8 | 2 | 25% |
| 15 | /COM2 | 10 | 2 | 20% |
| 16 | /PRN | 12 | 2 | 17% |
| 17 | /RTC | 14 | 1 | 7% |
| 18 | /BUFEN | 16 | 7 | 44% |
| 19 | RTCAS | 16 | 1 | 6% |
| 20–22 | inputs | — | 0 | — |
| 23 | NC | 8 | 0 | spare |

Total: **15 product terms** used, substantial headroom for future additions.

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
Pin 1  = AEN;
Pin 2  = A0;
Pin 3  = A3;
Pin 4  = A4;
Pin 5  = A5;
Pin 6  = A6;
Pin 7  = A7;
Pin 8  = A8;
Pin 9  = A9;
Pin 10 = J_COM1;
Pin 11 = J_COM2;
Pin 13 = J_PRN;

/* Active-low enable jumpers (active when LOW = shorted to GND) */
Pin 22 = !EN_COM1;  /* active low input */
Pin 21 = !EN_COM2;
Pin 20 = !EN_PRN;

/* Outputs (active-low) */
Pin 14 = !COM1_CS;
Pin 15 = !COM2_CS;
Pin 16 = !PRN_CS;
Pin 17 = !RTC_CS;
Pin 18 = !BUF_EN;
Pin 19 = RTCAS;

/* COM1: 0x3F8 or 0x2F8 — differ in A8 */
/* Common: A9=1, A7=1, A6=1, A5=1, A4=1, A3=1 */
COM1_CS = !AEN & EN_COM1
         & A9 & (A8 $ !J_COM1) & A7 & A6 & A5 & A4 & A3;

/* COM2: 0x3E8 or 0x2E8 — differ in A8 */
/* Common: A9=1, A7=1, A6=1, A5=1, A4=0, A3=1 */
COM2_CS = !AEN & EN_COM2
         & A9 & (A8 $ !J_COM2) & A7 & A6 & A5 & !A4 & A3;

/* PRN: 0x378 or 0x278 — differ in A8 */
/* Common: A9=1, A7=0, A6=1, A5=1, A4=1, A3=1 */
PRN_CS = !AEN & EN_PRN
       & A9 & (A8 $ !J_PRN) & !A7 & A6 & A5 & A4 & A3;

/* RTC: 0x70-0x71 (fixed) */
/* A9=0, A8=0, A7=0, A6=1, A5=1, A4=1, A3=0 */
RTC_CS = !AEN & !A9 & !A8 & !A7 & A6 & A5 & A4 & !A3;

/* Buffer enable: active when any CS is active */
BUF_EN = COM1_CS # COM2_CS # PRN_CS # RTC_CS;

/* RTC Address Strobe: AS = !A0 (inverted A0) */
RTCAS = !A0;
```
