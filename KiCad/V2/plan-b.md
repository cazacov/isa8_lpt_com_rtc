# ISA8 RTC Prototype Board - Plan B

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

Active signals on the prototype (directly active-low directly active-low directly active low directly active-low directly active low directly active-low directly active low directly) — only the subset needed for RTC:

| ISA Pin (component side) | Signal | Direction | Connection |
|---|---|---|---|
| B2 | D7 | Bidir | 74HCT245 pin 2 (A1) |
| B3 | D6 | Bidir | 74HCT245 pin 3 (A2) |
| B4 | D5 | Bidir | 74HCT245 pin 4 (A3) |
| B5 | D4 | Bidir | 74HCT245 pin 5 (A4) |
| B6 | D3 | Bidir | 74HCT245 pin 6 (A5) |
| B7 | D2 | Bidir | 74HCT245 pin 7 (A6) |
| B8 | D1 | Bidir | 74HCT245 pin 8 (A7) |
| B9 | D0 | Bidir | 74HCT245 pin 9 (A8) |
| A31 | A0 | In | ATF22V10 pin 2 |
| A30 | A1 | In | (not used — no connection) |
| A29 | A2 | In | (not used — no connection) |
| A28 | A3 | In | ATF22V10 pin 3 |
| A27 | A4 | In | ATF22V10 pin 4 |
| A26 | A5 | In | ATF22V10 pin 5 |
| A25 | A6 | In | ATF22V10 pin 6 |
| A24 | A7 | In | ATF22V10 pin 7 |
| A23 | A8 | In | ATF22V10 pin 8 |
| A22 | A9 | In | ATF22V10 pin 9 |
| B13 | IOW# | In | DS12885 pin 15 (R/W) |
| B14 | IOR# | In | DS12885 pin 17 (DS), 74HCT245 pin 1 (DIR) |
| A11 | AEN | In | ATF22V10 pin 1 |
| B4... | RESET DRV | In | DS12885 pin 18 (RESET) |
| B21 | IRQ7 | Out | JP1 option |
| B22 | IRQ6 | Out | JP1 option |
| B23 | IRQ5 | Out | JP1 option |
| B1 | GND | Power | Ground |
| B3/B29 | +5V | Power | VCC |

Note: ISA pin numbering follows standard XT 8-bit convention.

---

## 3. ATF22V10 Wiring on Prototype

Same firmware as plan.md. All pins are wired — some go to real signals, others to fixed levels.

| ATF22V10 Pin | Signal | Prototype Connection |
|---|---|---|
| 1 | AEN | ISA bus AEN |
| 2 | A0 | ISA bus A0 |
| 3 | A3 | ISA bus A3 |
| 4 | A4 | ISA bus A4 |
| 5 | A5 | ISA bus A5 |
| 6 | A6 | ISA bus A6 |
| 7 | A7 | ISA bus A7 |
| 8 | A8 | ISA bus A8 |
| 9 | A9 | ISA bus A9 |
| 10 | J_UART1 | Wire directly to GND (no UART1) |
| 11 | J_UART2 | Wire directly to GND (no UART2) |
| 12 | GND | Ground |
| 13 | J_PRN | Wire directly to GND (no PRN) |
| **14** | **UART1_CS#** | **No connection** (output floats, no UART1 chip) |
| **15** | **UART2_CS#** | **No connection** (output floats, no UART2 chip) |
| **16** | **PRN_CS#** | **No connection** (output floats, no PRN chip) |
| **17** | **RTC_CS#** | **DS12885 pin 13 (CS)** |
| **18** | **BUF_EN#** | **74HCT245 pin 19 (G#)** |
| **19** | **RTC_AS** | **DS12885 pin 14 (AS)** |
| 20 | EN_PRN# | Wire directly to VCC (PRN disabled) |
| 21 | EN_UART2# | Wire directly to VCC (UART2 disabled) |
| 22 | EN_UART1# | Wire directly to VCC (UART1 disabled) |
| **23** | **(spare)** | **No connection** |
| 24 | VCC | +5V |

Because EN_UART1#, EN_UART2#, and EN_PRN# are pulled HIGH (disabled), the PLD
will never assert UART1_CS#, UART2_CS#, or PRN_CS#. BUF_EN# activates only when
RTC_CS# fires.

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
| 13 | CS | ATF22V10 pin 17 (RTC_CS#) |
| 14 | AS | ATF22V10 pin 19 (RTC_AS = !A0) |
| 15 | R/W | ISA bus IOW# (write-enable, data latched on rising edge) |
| 16 | GND | Ground (second GND pad) |
| 17 | DS | ISA bus IOR# (output-enable for reads) |
| 18 | RESET | ISA bus RESET DRV |
| 19 | IRQ | R1 (4.7k pull-up to VCC), then to JP1 pin 1 |
| 20 | VBAT | BT1 positive (+3V CR2032) |
| 21 | RCLR | No connection (has internal pull-up per datasheet) |
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
  D0─D7          A0  A3─A9      IOR#   IOW#   AEN   RESET   IRQ5  IRQ6  IRQ7
   │              │   │           │      │      │      │       │     │     │
   │   ┌──────────┤   │           │      │      │      │       │     │     │
   │   │          │   │           │      │      │      │       │     │     │
   │   │          │   │           │      │      │      │       │     │     │
═══╪═══╪══════════╪═══╪═══════════╪══════╪══════╪══════╪═══════╪═════╪═════╪════
   │   │          │   │           │      │      │      │       │     │     │
   │   │          │   │           │      │      │      │     ┌─┴──┬──┴──┬──┘
   │   │          │   │           │      │      │      │     │ JP1│     │
   │   │          │   │           │      │      │      │     │2  3│  4  │
   │   │          │   │           │      │      │      │     └─┬──┴──┬──┘
   │   │          │   │           │      │      │      │       └──┬──┘
   │   │          │   │           │      │      │      │          │1
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
│  │ Card Side:         │   │ pin 10 J1  ─┤10k├─GND  │ pin 8  AD4 ─┼─┼──pin 14 B5
│  │ pin 18 B1 ── AD0   │   │ pin 11 J2  ─┤10k├─GND  │ pin 9  AD5 ─┼─┼──pin 13 B6
│  │ pin 17 B2 ── AD1   │   │ pin 13 JP  ─┤10k├─GND  │ pin 10 AD6 ─┼─┼──pin 12 B7
│  │ pin 16 B3 ── AD2   │   │                      │   │ pin 11 AD7 ─┼─┼──pin 11 B8
│  │ pin 15 B4 ── AD3   │   │ pin 22 EU1 ─┤10k├─VCC  │              │ │  │
│  │ pin 14 B5 ── AD4   │   │ pin 21 EU2 ─┤10k├─VCC  │ pin 12 GND──┼─┼──GND
│  │ pin 13 B6 ── AD5   │   │ pin 20 EP  ─┤10k├─VCC  │ pin 13 CS ──┼─┼──────────┐
│  │ pin 12 B7 ── AD6   │   │                      │   │ pin 14 AS ──┼─┼────────┐ │
│  │ pin 11 B8 ── AD7   │   │ Outputs:             │   │ pin 15 R/W──┼─┼─ IOW#  │ │
│  │                    │   │ pin 14 UART1_CS#  NC │   │ pin 16 GND──┼─┼──GND   │ │
│  │ pin 1  DIR ── IOR# │   │ pin 15 UART2_CS#  NC │   │ pin 17 DS ──┼─┼─ IOR#  │ │
│  │ pin 19 G#  ────────┼───┤ pin 16 PRN_CS#    NC │   │ pin 18 RST──┼─┼─RESET  │ │
│  │ pin 10 GND ── GND  │   │ pin 17 RTC_CS# ─────┼───┤►pin 13 CS   │ │        │ │
│  │ pin 20 VCC ── VCC  │   │ pin 18 BUF_EN# ─────┼───┤►pin 19 G#   │ │        │ │
│  │                    │   │ pin 19 RTC_AS  ──────┼───┤►pin 14 AS   │ │        │ │
│  └────────────────────┘   │ pin 23 (spare)    NC │   │ pin 19 IRQ──┼─┼─ JP1   │ │
│                           │                      │   │ pin 20 VBAT─┼─┼─ BT1+  │ │
│                           │ pin 12 GND ── GND    │   │ pin 21 RCLR─┤10k├─VCC  │ │
│                           │ pin 24 VCC ── VCC    │   │ pin 22 N.C. │ │        │ │
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
ISA_D0          ISA D0 (B9)             U2 pin 9 (A8)
ISA_D1          ISA D1 (B8)             U2 pin 8 (A7)
ISA_D2          ISA D2 (B7)             U2 pin 7 (A6)
ISA_D3          ISA D3 (B6)             U2 pin 6 (A5)
ISA_D4          ISA D4 (B5)             U2 pin 5 (A4)
ISA_D5          ISA D5 (B4)             U2 pin 4 (A3)
ISA_D6          ISA D6 (B3)             U2 pin 3 (A2)
ISA_D7          ISA D7 (B2)             U2 pin 2 (A1)
CARD_D0         U2 pin 18 (B1)          U3 pin 4 (AD0)
CARD_D1         U2 pin 17 (B2)          U3 pin 5 (AD1)
CARD_D2         U2 pin 16 (B3)          U3 pin 6 (AD2)
CARD_D3         U2 pin 15 (B4)          U3 pin 7 (AD3)
CARD_D4         U2 pin 14 (B5)          U3 pin 8 (AD4)
CARD_D5         U2 pin 13 (B6)          U3 pin 9 (AD5)
CARD_D6         U2 pin 12 (B7)          U3 pin 10 (AD6)
CARD_D7         U2 pin 11 (B8)          U3 pin 11 (AD7)
ISA_A0          ISA A0 (A31)            U1 pin 2
ISA_A3          ISA A3 (A28)            U1 pin 3
ISA_A4          ISA A4 (A27)            U1 pin 4
ISA_A5          ISA A5 (A26)            U1 pin 5
ISA_A6          ISA A6 (A25)            U1 pin 6
ISA_A7          ISA A7 (A24)            U1 pin 7
ISA_A8          ISA A8 (A23)            U1 pin 8
ISA_A9          ISA A9 (A22)            U1 pin 9
ISA_AEN         ISA AEN (A11)           U1 pin 1
ISA_IOR#        ISA IOR# (B14)          U2 pin 1 (DIR), U3 pin 17 (DS)
ISA_IOW#        ISA IOW# (B13)          U3 pin 15 (R/W)
ISA_RESET       ISA RESET DRV (B2)      U3 pin 18 (RESET)
RTC_CS#         U1 pin 17               U3 pin 13 (CS)
BUF_EN#         U1 pin 18               U2 pin 19 (G#)
RTC_AS          U1 pin 19               U3 pin 14 (AS)
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
  1   CPU drives A0=0,        AEN=0              (waiting)         (waiting)
      A3-A9 = 0001110
      AEN=0

  2   PLD decodes             RTC_CS# → LOW      G# goes LOW      CS asserted
      port 0x70               BUF_EN# → LOW      (buffer on)
                              RTC_AS → HIGH
                              (because !A0=1)

  3   CPU drives data         (stable)           DIR=1 (IOR#=1)   AS=HIGH, sees
      on D0-D7                                   A→B: ISA→card    address on AD0-7

  4   IOW# goes LOW           (stable)           (stable)          R/W goes LOW

  5   IOW# returns HIGH       (stable)           (stable)          R/W rising edge
                                                                   latches AD0-7
                                                                   as register addr

  6   Bus cycle ends           CS#→HIGH           G#→HIGH          CS deasserted
                               AS→LOW
```

### Test 2: Read RTC Data Register (IN AL, 71h)

```
Step  ISA Bus                 ATF22V10           74HCT245         DS12885
────  ────────────────────    ─────────────────  ────────────────  ──────────────
  1   CPU drives A0=1,        AEN=0              (waiting)         (waiting)
      A3-A9 = 0001110
      AEN=0

  2   PLD decodes             RTC_CS# → LOW      G# goes LOW      CS asserted
      port 0x71               BUF_EN# → LOW      (buffer on)
                              RTC_AS → LOW
                              (because !A0=0)

  3   IOR# goes LOW           (stable)           DIR=0 (IOR#=0)   DS goes LOW
                                                 B→A: card→ISA    RTC drives data
                                                                   onto AD0-7

  4   Data flows:             (stable)           AD0-7 → D0-7     (driving bus)
      DS12885 → 74245 B→A → ISA D0-D7
      CPU reads D0-D7

  5   IOR# returns HIGH       (stable)           DIR=1, G#→HIGH   DS goes HIGH
      Bus cycle ends           CS#→HIGH                            bus released
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
- [ ] PLD decode: use logic analyzer on RTC_CS# (U1-17) — should pulse LOW only
      during port 0x70 or 0x71 access
- [ ] PLD non-decode: verify UART1_CS#, UART2_CS#, PRN_CS# never go LOW
      (EN pulled high = disabled)
- [ ] BUF_EN#: should match RTC_CS# timing (only RTC active on prototype)
- [ ] RTC_AS: should be HIGH during port 0x70 access, LOW during port 0x71
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
