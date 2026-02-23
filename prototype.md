# ISA8 Prototype Board - Test RTC signals decoding

## Goal

Minimal ISA 8-bit card to validate:
1. ATF22V10 address decode logic (same firmware as full board)
2. 74HCT245 bus buffering
3. DS12885 RTC operation at ports 0x70/0x71

UART and PRN chips are not populated. Their ATF22V10 inputs are tied to safe defaults;
their ATF22V10 outputs are left unconnected.

---

## 1. Components

| Ref | Component | Package | Qty | Notes |
|---|---|---|---|---|
| U1 | ATF22V10C-15PU | DIP-24 | 1 | Address decode (full firmware) |
| U2 | 74HCT245 | DIP-20 | 1 | Bus buffer |
| U3 | DS12885 | DIP-24 | 1 | Real-time clock |
| Y1 | 32.768 kHz crystal | HC49 | 1 | RTC oscillator (6 pF load) |
| BT1 | CR2032 holder + cell | Through-hole | 1 | RTC battery backup |
| J1 | ISA 8-bit edge connector | 62-pin (2x31) | 1 | Bus interface |
| JP1 | 1x4 pin header | 2.54 mm | 1 | RTC IRQ select |
| R1 | 4.7k resistor | Axial | 1 | Pull-up for DS12885 IRQ (open-drain). Optional if IRQ not used. |
| C1 | 100 nF ceramic | Axial/radial | 1 | Bypass cap, ATF22V10 |
| C2 | 100 nF ceramic | Axial/radial | 1 | Bypass cap, 74HCT245 |
| C3 | 100 nF ceramic | Axial/radial | 1 | Bypass cap, DS12885 |
| C4 | 10 uF electrolytic | Radial | 1 | Bulk decoupling, VCC |

---

## 2. ISA Bus Signals Used

Only the subset of ISA bus signals needed for RTC operation:

| ISA Pin | Signal | Direction | Connection |
|---|---|---|---|
| A9 | D0 | Bidir | 74HCT245 pin 2 (A1) |
| A8 | D1 | Bidir | 74HCT245 pin 3 (A2) |
| A7 | D2 | Bidir | 74HCT245 pin 4 (A3) |
| A6 | D3 | Bidir | 74HCT245 pin 5 (A4) |
| A5 | D4 | Bidir | 74HCT245 pin 6 (A5) |
| A4 | D5 | Bidir | 74HCT245 pin 7 (A6) |
| A3 | D6 | Bidir | 74HCT245 pin 8 (A7) |
| A2 | D7 | Bidir | 74HCT245 pin 9 (A8) |
| A31 | A0 | In | ATF22V10 pin 1 |
| A30 | A1 | In | (not used — no connection) |
| A29 | A2 | In | ATF22V10 pin 2 |
| A28 | A3 | In | ATF22V10 pin 3 |
| A27 | A4 | In | ATF22V10 pin 4 |
| A26 | A5 | In | ATF22V10 pin 5 |
| A25 | A6 | In | ATF22V10 pin 6 |
| A24 | A7 | In | ATF22V10 pin 7 |
| A23 | A8 | In | ATF22V10 pin 8 |
| A22 | A9 | In | ATF22V10 pin 9 |
| B13 | IOW# | In | ATF22V10 pin 11 |
| B14 | IOR# | In | ATF22V10 pin 10, 74HCT245 pin 1 (DIR) |
| A11 | AEN | In | ATF22V10 pin 13 |
| B21 | IRQ7 | Out | JP1 option |
| B22 | IRQ6 | Out | JP1 option |
| B23 | IRQ5 | Out | JP1 option |
| B1, B10 | GND | Power | Ground |
| B3, B29 | +5V | Power | VCC |

Note: ISA pin numbering follows standard XT 8-bit convention.

---

## 3. ATF22V10 Wiring on Prototype

Same firmware as plan.md. All pins are wired — some go to real signals, others to fixed levels.

| ATF22V10 Pin | Signal | Prototype Connection |
|---|---|---|
| 1 | A0 | ISA bus A0 |
| 2 | A2 | ISA bus A2 |
| 3 | A3 | ISA bus A3 |
| 4 | A4 | ISA bus A4 |
| 5 | A5 | ISA bus A5 |
| 6 | A6 | ISA bus A6 |
| 7 | A7 | ISA bus A7 |
| 8 | A8 | ISA bus A8 |
| 9 | A9 | ISA bus A9 |
| 10 | IOR# | ISA bus IOR# (input to PLD) |
| 11 | IOW# | ISA bus IOW# (input to PLD) |
| 12 | GND | Ground |
| 13 | AEN | ISA bus AEN (DMA indicator) |
| **14** | **COM1_CS#** | **No connection** (output floats, no COM1 chip) |
| **15** | **COM2_CS#** | **No connection** (output floats, no COM2 chip) |
| **16** | **PRN_CS#** | **No connection** (output floats, no PRN chip) |
| **17** | **RTC_WR#** | **DS12885 pin 15 (WR# in Intel mode)** |
| **18** | **BUF_EN#** | **74HCT245 pin 19 (G#)** |
| **19** | **RTC_AS** | **DS12885 pin 14 (AS)** |
| **20** | **RTC_DS#** | **DS12885 pin 17 (DS# in Intel mode)** |
| 21 | J_COM1 | Wire directly to GND (no COM1) |
| 22 | J_COM2 | Wire directly to GND (no COM2) |
| 23 | J_PRN | Wire directly to GND (no PRN) |
| 24 | VCC | +5V |

IOR# and IOW# are on dedicated input pins 10–11. All three address jumpers
(J_COM1, J_COM2, J_PRN) are grouped on adjacent I/O pins 21–23, wired to GND
on this prototype board. A0 and A2 are on dedicated input pins 1–2 for compact
address bus routing; AEN is on I/O pin 13 (configured as input). The PLD
generates qualified RTC_WR#
and RTC_DS# strobes — DS12885 CS# is tied to GND (always selected). BUF_EN#
activates whenever any device address range is on the bus.

---

## 4. DS12885 Wiring (DIP-24, Intel Mode)

| DS12885 Pin | Name | Prototype Connection |
|---|---|---|
| 1 | MOT | No connection (internal 20k pull-down → Intel mode) |
| 2 | X1 | Y1 crystal leg 1 |
| 3 | X2 | Y1 crystal leg 2 |
| 4 | AD0 | 74HCT245 pin 18 (B1) |
| 5 | AD1 | 74HCT245 pin 17 (B2) |
| 6 | AD2 | 74HCT245 pin 16 (B3) |
| 7 | AD3 | 74HCT245 pin 15 (B4) |
| 8 | AD4 | 74HCT245 pin 14 (B5) |
| 9 | AD5 | 74HCT245 pin 13 (B6) |
| 10 | AD6 | 74HCT245 pin 12 (B7) |
| 11 | AD7 | 74HCT245 pin 11 (B8) |
| 12 | GND | Ground |
| 13 | CS | GND (always selected — strobes are PLD-qualified) |
| 14 | AS | ATF22V10 pin 19 (RTC_AS) |
| 15 | WR# | ATF22V10 pin 17 (RTC_WR# — qualified write strobe for 0x71) |
| 16 | GND | Ground (second GND pad) |
| 17 | DS# | ATF22V10 pin 20 (RTC_DS# — qualified data strobe for 0x71) |
| 18 | RESET | VCC (per datasheet: allows power-fail transitions without clearing control registers) |
| 19 | IRQ | R1 (4.7k pull-up to VCC), then to JP1 pin 1 |
| 20 | VBAT | BT1 positive (+3V CR2032) |
| 21 | RCLR | Internal pull-up; 2-pin header to GND for manual CMOS clear (optional on prototype) |
| 22 | N.C. | No connection |
| 23 | SQW | No connection (or route to test point) |
| 24 | VCC | +5V |

---

## 5. 74HCT245 Wiring (DIP-20)

| 74HCT245 Pin | Name | Prototype Connection |
|---|---|---|
| 1 | DIR | ISA bus IOR# |
| 2 | A1 | ISA bus D0 |
| 3 | A2 | ISA bus D1 |
| 4 | A3 | ISA bus D2 |
| 5 | A4 | ISA bus D3 |
| 6 | A5 | ISA bus D4 |
| 7 | A6 | ISA bus D5 |
| 8 | A7 | ISA bus D6 |
| 9 | A8 | ISA bus D7 |
| 10 | GND | Ground |
| 11 | B8 | DS12885 pin 11 (AD7) |
| 12 | B7 | DS12885 pin 10 (AD6) |
| 13 | B6 | DS12885 pin 9 (AD5) |
| 14 | B5 | DS12885 pin 8 (AD4) |
| 15 | B4 | DS12885 pin 7 (AD3) |
| 16 | B3 | DS12885 pin 6 (AD2) |
| 17 | B2 | DS12885 pin 5 (AD1) |
| 18 | B1 | DS12885 pin 4 (AD0) |
| 19 | G# | ATF22V10 pin 18 (BUF_EN#) |
| 20 | VCC | +5V |

Direction logic:
- IOR# = 0 → DIR = 0 → B-to-A (RTC drives data to ISA bus)
- IOR# = 1 → DIR = 1 → A-to-B (ISA bus writes to RTC)

---

## 6. RTC IRQ Jumper (JP1)

```
JP1:  1x4 pin header

  Pin 1 ── R1 (4.7k) ── VCC     ← RTC IRQ (open-drain, active low)
           │
           DS12885 pin 19

  Pin 2 ── ISA IRQ5 (B23)
  Pin 3 ── ISA IRQ6 (B22)       ← not commonly used, good for testing
  Pin 4 ── ISA IRQ7 (B21)

  Jumper wire from Pin 1 to one of Pin 2/3/4 selects the IRQ line.
  Leave open if no IRQ needed (RTC still works for polled access).
```

---

## 7. Circuit Diagram

```
═══════════════════════════ ISA 8-bit Bus Edge Connector (J1) ══════════════════
  D0─D7          A0  A3─A9      IOR#   IOW#   AEN             IRQ5  IRQ6  IRQ7
   │              │   │           │      │      │               │     │     │
   │   ┌──────────┤   │           │      │      │               │     │     │
   │   │          │   │           │      │      │               │     │     │
   │   │          │   │           │      │      │               │     │     │
═══╪═══╪══════════╪═══╪═══════════╪══════╪══════╪═══════════════╪═════╪═════╪════
   │   │          │   │           │      │      │               │     │     │
   │   │          │   │           │      │      │             ┌─┴──┬──┴──┬──┘
   │   │          │   │           │      │      │             │ JP1│     │
   │   │          │   │           │      │      │             │2  3│  4  │
   │   │          │   │           │      │      │             └─┬──┴──┬──┘
   │   │          │   │           │      │      │               └──┬──┘
   │   │          │   │           │      │      │                 │1
   │   │          │   │           │      │      │      │          │
   │   │          │   │           │      │      │      │      R1 4.7k
   │   │          │   │           │      │      │      │          │
   │   │          │   │           │      │      │      │         VCC
   │   │          │   │           │      │      │      │
   │   │  ┌───────┘   │           │      │      │      │
   │   │  │  ┌────────┘           │      │      │      │
   │   │  │  │                    │      │      │      │
   │   │  │  │    ┌───────────────┤      │      │      │
   │   │  │  │    │    ┌──────────┼──────┤      │      │
   │   │  │  │    │    │          │      │      │      │
   │   │  │  │    │    │          │      │      │      │
   │   │  │  │    │    │          │      │      │      │
   ▼   ▼  ▼  ▼    ▼    ▼          ▼      ▼      ▼      ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│  ┌────────────────────┐   ┌──────────────────────┐   ┌────────────────┐  │
│  │    U2: 74HCT245    │   │    U1: ATF22V10      │   │  U3: DS12885   │  │
│  │    (DIP-20)        │   │    (DIP-24)          │   │  (DIP-24)      │  │
│  │                    │   │                      │   │                │  │
│  │ ISA Side:          │   │ Inputs:              │   │ Pin 1  MOT  NC│  │
│  │ pin 2  A1 ── D0    │   │ pin 1  AEN ── AEN   │   │ pin 2  X1 ──┐ │  │
│  │ pin 3  A2 ── D1    │   │ pin 2  A0  ── A0    │   │ pin 3  X2 ──┤ │  │
│  │ pin 4  A3 ── D2    │   │ pin 3  A3  ── A3    │   │        Y1   │ │  │
│  │ pin 5  A4 ── D3    │   │ pin 4  A4  ── A4    │   │       32768Hz│ │  │
│  │ pin 6  A5 ── D4    │   │ pin 5  A5  ── A5    │   │              │ │  │
│  │ pin 7  A6 ── D5    │   │ pin 6  A6  ── A6    │   │ pin 4  AD0 ─┼─┼──pin 18 B1
│  │ pin 8  A7 ── D6    │   │ pin 7  A7  ── A7    │   │ pin 5  AD1 ─┼─┼──pin 17 B2
│  │ pin 9  A8 ── D7    │   │ pin 8  A8  ── A8    │   │ pin 6  AD2 ─┼─┼──pin 16 B3
│  │                    │   │ pin 9  A9  ── A9    │   │ pin 7  AD3 ─┼─┼──pin 15 B4
│  │ Card Side:         │   │ pin 10 IOR ── IOR#    │ pin 8  AD4 ─┼─┼──pin 14 B5
│  │ pin 18 B1 ── AD0   │   │ pin 11 IOW ── IOW#    │ pin 9  AD5 ─┼─┼──pin 13 B6
│  │ pin 17 B2 ── AD1   │   │ pin 13 NC             │   │ pin 10 AD6 ─┼─┼──pin 12 B7
│  │ pin 16 B3 ── AD2   │   │                      │   │ pin 11 AD7 ─┼─┼──pin 11 B8
│  │ pin 15 B4 ── AD3   │   │ pin 23 JP  ─┤10k├─GND  │   │              │ │  │
│  │ pin 14 B5 ── AD4   │   │ pin 22 J2  ─┤10k├─GND  │ pin 12 GND──┼─┼──GND
│  │ pin 13 B6 ── AD5   │   │ pin 21 J1  ─┤10k├─GND  │ pin 13 CS ──┼─┼──GND
│  │ pin 12 B7 ── AD6   │   │ pin 20 RTC_DS#────────┼───┤►pin 17 DS#   │ │
│  │ pin 11 B8 ── AD7   │   │ Outputs:             │   │ pin 14 AS   │ │        │
│  │                    │   │ pin 14 COM1_CS#  NC │   │ pin 15 WR#  │ │        │
│  │ pin 1  DIR ── IOR# │   │ pin 15 COM2_CS#  NC │   │ pin 16 GND──┼─┼──GND   │
│  │ pin 19 G#  ────────┼───┤ pin 16 PRN_CS#    NC │   │ pin 17 DS#   │ │        │
│  │ pin 10 GND ── GND  │   │ pin 17 RTC_WR# ──────┼───┤►pin 15 WR#  │ │        │
│  │ pin 20 VCC ── VCC  │   │ pin 18 BUF_EN# ─────┼───┤►pin 19 G#   │ │        │
│  │                    │   │ pin 19 RTC_AS  ──────┼───┤►pin 14 AS   │ │        │
│  └────────────────────┘   │ pin 20 RTC_DS#  ─────┼───┤►pin 17 DS#   │ │        │
│                           │                      │   │ pin 18 RST──┼─┼─VCC    │
│                           │ pin 12 GND ── GND    │   │ pin 19 IRQ──┼─┼─ JP1   │
│                           │ pin 24 VCC ── VCC    │   │ pin 20 VBAT─┼─┼─ BT1+  │
│                           │                      │   │ pin 23 SQW  NC│        │ │
│                           └──────────────────────┘   │ pin 24 VCC──┼─┼──VCC   │ │
│                                                      │             │ │        │ │
│                                                      └─────────────┘ │        │ │
│                                                                      │        │ │
│  Bypass capacitors: C1 (100nF) U1 VCC-GND                           │        │ │
│                     C2 (100nF) U2 VCC-GND                           │        │ │
│                     C3 (100nF) U3 VCC-GND                           │        │ │
│                     C4 (10uF)  bulk VCC-GND near ISA connector      │        │ │
│                                                                      │        │ │
└──────────────────────────────────────────────────────────────────────┘        │ │
                                                                                │ │
                            (arrows show signal flow, all active connections)   │ │
```

### Detailed Net List

```
Net Name        From                    To
────────────────────────────────────────────────────────
ISA_D0          ISA D0 (A9)             U2 pin 2 (A1)
ISA_D1          ISA D1 (A8)             U2 pin 3 (A2)
ISA_D2          ISA D2 (A7)             U2 pin 4 (A3)
ISA_D3          ISA D3 (A6)             U2 pin 5 (A4)
ISA_D4          ISA D4 (A5)             U2 pin 6 (A5)
ISA_D5          ISA D5 (A4)             U2 pin 7 (A6)
ISA_D6          ISA D6 (A3)             U2 pin 8 (A7)
ISA_D7          ISA D7 (A2)             U2 pin 9 (A8)
CARD_D0         U2 pin 18 (B1)          U3 pin 4 (AD0)
CARD_D1         U2 pin 17 (B2)          U3 pin 5 (AD1)
CARD_D2         U2 pin 16 (B3)          U3 pin 6 (AD2)
CARD_D3         U2 pin 15 (B4)          U3 pin 7 (AD3)
CARD_D4         U2 pin 14 (B5)          U3 pin 8 (AD4)
CARD_D5         U2 pin 13 (B6)          U3 pin 9 (AD5)
CARD_D6         U2 pin 12 (B7)          U3 pin 10 (AD6)
CARD_D7         U2 pin 11 (B8)          U3 pin 11 (AD7)
ISA_A0          ISA A0 (A31)            U1 pin 1
ISA_A2          ISA A2 (A29)            U1 pin 2
ISA_A3          ISA A3 (A28)            U1 pin 3
ISA_A4          ISA A4 (A27)            U1 pin 4
ISA_A5          ISA A5 (A26)            U1 pin 5
ISA_A6          ISA A6 (A25)            U1 pin 6
ISA_A7          ISA A7 (A24)            U1 pin 7
ISA_A8          ISA A8 (A23)            U1 pin 8
ISA_A9          ISA A9 (A22)            U1 pin 9
ISA_AEN         ISA AEN (A11)           U1 pin 13
ISA_IOR#        ISA IOR# (B14)          U2 pin 1 (DIR), U1 pin 10 (IOR)
ISA_IOW#        ISA IOW# (B13)          U1 pin 11 (IOW)
RTC_RESET       VCC                     U3 pin 18 (RESET)
RTC_WR#         U1 pin 17               U3 pin 15 (WR#)
BUF_EN#         U1 pin 18               U2 pin 19 (G#)
RTC_AS          U1 pin 19               U3 pin 14 (AS)
RTC_DS#         U1 pin 20               U3 pin 17 (DS#)
RTC_CS#         GND                     U3 pin 13 (CS)
RTC_IRQ         U3 pin 19               R1 (4.7k to VCC), JP1 pin 1
VCC             ISA +5V                 U1-24, U2-20, U3-24, R pull-ups, C bypass
GND             ISA GND                 U1-12, U2-10, U3-12, U3-16, R pull-downs, C bypass
VBAT            BT1 (+)                 U3 pin 20
```

---

## 8. Bus Cycle Walkthrough (Prototype Verification)

### Test 1: Write RTC Address Register (OUT 70h, value)

```
Step  ISA Bus                 ATF22V10           74HCT245         DS12885
────  ────────────────────    ─────────────────  ────────────────  ──────────────
  1   CPU drives A0=0,        AEN=0              (waiting)         CS=GND (always
      A3-A9 = 0001110                                              selected)
      AEN=0

  2   PLD decodes             BUF_EN# → LOW      G# goes LOW      WR# HIGH (idle)
      port 0x70               RTC_AS still LOW   (buffer on)      DS# HIGH (idle)
                              RTC_WR# HIGH
                              RTC_DS# HIGH
                              (IOW#/IOR# not
                               yet active)

  3   CPU drives data         (stable)           DIR=1 (IOR#=1)   AS still LOW
      on D0-D7                                   A→B: ISA→card    data on AD0-7

  4   IOW# goes LOW           RTC_AS → HIGH      (stable)          AS goes HIGH
                              (addr match +                        → latch opens
                               A0=0 + IOW#)                       (transparent)
                              RTC_WR# stays HIGH                   WR# stays HIGH
                              (A0=0, no data                      (no data write)
                               write)

  5   IOW# returns HIGH       RTC_AS → LOW       (stable)          AS falls →
                                                                   address latch
                                                                   FREEZES,
                                                                   capturing
                                                                   reg addr.

  6   Bus cycle ends           BUF_EN#→HIGH      G#→HIGH          Latch frozen,
      Between cycles,          RTC_AS stays LOW                    address safe.
      RTC_AS stays LOW —       (addr mismatch)                     WR#/DS# idle.
      no corruption possible
```

### Test 2: Read RTC Data Register (IN AL, 71h)

```
Step  ISA Bus                 ATF22V10           74HCT245         DS12885
────  ────────────────────    ─────────────────  ────────────────  ──────────────
  1   CPU drives A0=1,        AEN=0              (waiting)         CS=GND (always
      A3-A9 = 0001110                                              selected)
      AEN=0

  2   PLD decodes             BUF_EN# → LOW      G# goes LOW      WR# HIGH (idle)
      port 0x71               RTC_AS stays LOW   (buffer on)      DS# HIGH (idle)
                              (A0=1)

  3   IOR# goes LOW           RTC_DS# → LOW       DIR=0 (IOR#=0)   DS# goes LOW
                              (addr match +      B→A: card→ISA    → RTC drives
                               A0=1 + IOR#)                        data onto
                                                                   AD0-AD7

  4   Data flows:             (stable)           AD0-7 → D0-7     (driving bus)
      DS12885 → 74245 B→A → ISA D0-D7
      CPU reads D0-D7

  5   IOR# returns HIGH       RTC_DS# → HIGH      DIR=1, G#→HIGH   DS# goes HIGH
      Bus cycle ends           BUF_EN#→HIGH                        bus released
```

### Test 3: Write RTC Data Register (OUT 71h, value)

```
Step  ISA Bus                 ATF22V10           74HCT245         DS12885
────  ────────────────────    ─────────────────  ────────────────  ──────────────
  1   CPU drives A0=1,        AEN=0              (waiting)         CS=GND
      A3-A9 = 0001110
      AEN=0

  2   PLD decodes             BUF_EN# → LOW      G# goes LOW      WR# HIGH
      port 0x71               RTC_AS stays LOW   (buffer on)      DS# HIGH
                              (A0=1)

  3   CPU drives write data   (stable)           DIR=1 (IOR#=1)   (waiting)
      on D0-D7                                   A→B: ISA→card

  4   IOW# goes LOW           RTC_WR# → LOW       (stable)          WR# goes LOW
                              (addr match +                        (write mode)
                               A0=1 + IOW#)

  5   IOW# returns HIGH       RTC_WR# → HIGH      (stable)          WR# rising edge
                                                                   → DS12885
                                                                   latches data
                                                                   from AD0-AD7

  6   Bus cycle ends           BUF_EN#→HIGH      G#→HIGH          Write complete
```

---

## 9. Software Test Program (DOS DEBUG)

```
; Set register address to 0x00 (seconds)
O 70 00

; Read seconds value
I 71
; Should return BCD seconds (00-59) or binary depending on DM bit

; Read register 0x0A (Register A) - check oscillator
O 70 0A
I 71
; Bit 7 (UIP) toggles once per second; bits 6-4 should be 010 (DV=010 = oscillator on)
; Expected value when running: 0x26 (DV=010, RS=0110 = 1024 Hz default)

; Write Register B to set 24-hour BCD mode
O 70 0B
O 71 02
; Sets 24-hour mode (bit 1=1), BCD (bit 2=0)

; Verify Register D (valid RAM bit)
O 70 0D
I 71
; Bit 7 should be 1 if battery is good (VRT=1)
```

---

## 10. Prototype Test Checklist

- [ ] Power: verify +5V on U1-24, U2-20, U3-24 and GND continuity
- [ ] No bus contention: with card inserted but before testing, check no data bus
      pins are being driven (74245 G# should be HIGH when no I/O cycle)
- [ ] PLD decode: use logic analyzer on RTC_WR# (U1-17) and RTC_DS# (U1-20) —
      RTC_WR# should pulse LOW only during OUT 71h; RTC_DS# only during IN 71h
- [ ] PLD non-decode: verify COM1_CS#, COM2_CS#, PRN_CS# never go LOW
- [ ] BUF_EN#: should go LOW during port 0x70 and 0x71 access
- [ ] DS12885 CS# (pin 13): verify wired to GND
- [ ] RTC_AS: should pulse HIGH only during IOW# active phase of port 0x70
      write; should stay LOW during port 0x71 access and all non-RTC cycles
- [ ] Direction: verify 74245 DIR follows IOR# correctly
- [ ] RTC read: `I 71` after `O 70 00` returns changing seconds value
- [ ] RTC write: can set time, read back matches
- [ ] Battery backup: remove power, wait, reinsert — time should persist
- [ ] IRQ (optional): program RTC periodic interrupt, verify interrupt fires on selected IRQ

---

## 11. Known Limitations of Prototype

1. **No UART/PRN** — only RTC is testable
2. **PLD outputs 14-16 float** — acceptable since nothing is connected; they are
   driven by the PLD (not tri-stated), so they won't cause issues
3. **Crystal layout matters** — keep Y1 traces short, away from data bus traces,
   with local ground plane under the crystal if possible
4. **ISA bus loading** — single 74HCT245 presents minimal load; no concerns
5. **Battery**: do NOT install CR2032 backwards — no reverse polarity protection
   in this minimal design (consider adding a diode in the full board)
