# ISA8 LPT COM RTC

An 8-bit ISA expansion card providing a parallel port, two serial ports, and a battery-backed real-time clock for retro PC/XT systems.

## Features

- **Parallel Port (PRN)** — UM82C11-C, DB-25 connector, Centronics-compatible
- **2 × Serial Port (COM)** — 16450/16550-compatible UARTs, RS-232 levels via GD75232N
- **Real-Time Clock (RTC)** — DS12885 with CR2032 battery backup and CMOS RAM
- **Single-chip address decoding** — ATF22V10 PLD, no additional logic gates
- **Fully jumper-configurable** — port addresses and IRQs
- **Socket-based device enable/disable** — devices disabled by removing chip from socket; 10k pull-up resistor network on data bus ensures clean 0xFF reads for empty sockets

## Target Systems

- IBM PC/XT and compatibles (8088/8086)
- Compatible with [8088 BIOS](https://github.com/skiselev/8088_bios)
- DOS and early Windows

## Design Specifications

### ISA Bus Interface

- 8-bit data bus (D0–D7)
- Standard ISA 8-bit (62-pin) edge connector
- +5 V, +12 V, −12 V power rails from bus

### Parallel Port (PRN)

- UM82C11-C printer adapter interface chip
- DB-25 connector, standard Centronics-compatible
- Configurable base address: 0x378 (LPT1) or 0x278 (LPT2)
- Configurable interrupt: IRQ7 or IRQ5
- Active-low chip select: PRN_CS#

### Serial Ports (UART)

Both ports use 16450/16550-compatible UARTs with RS-232 level shifting (GD75232N or equivalent). CS0 and CS1 are tied to GND; CS2 is the active-low chip select driven by the PLD.

| Port | Connector | Base Address Options | IRQ Options | CS Signal |
|------|-----------|---------------------|-------------|-----------|
| COM1 | DB-9 | 0x3F8 or 0x2F8 | IRQ4 or IRQ3 | COM1_CS# |
| COM2 | 2×5 header | 0x3E8 or 0x2E8 | IRQ4 or IRQ3 | COM2_CS# |

### Real-Time Clock (RTC)

- DS12885 RTC chip with CMOS RAM for BIOS settings
- CR2032 battery backup
- Fixed I/O ports 0x70 (address) / 0x71 (data)
- Intel bus timing (MOT pin left unconnected; internal 20kΩ pull-down selects Intel mode)
- Configurable interrupt: IRQ2/9, IRQ5, IRQ6, or IRQ7
- IRQ polarity inversion: DS12885 IRQ output is active-low open-drain; an NPN transistor (Q1, 2N3904) with a 330Ω series current-limiting resistor converts it to the active-high signal expected by the ISA bus 8259A PIC (see [plan.md](plan.md) Section 6 for circuit details)
- On PC/XT (single 8259A), the IRQ2 jumper position triggers IRQ2 (INT 0Ah); on AT-class systems (cascaded 8259A), the same physical IRQ2 line is routed to the slave PIC as IRQ9 (INT 71h), which the BIOS redirects to INT 0Ah for compatibility

#### Multiplexed Address/Data Bus and Address Latching

The DS12885 uses a **multiplexed** address/data bus (AD0–AD7). The same eight pins carry a 7-bit internal register address (0x00–0x7F) during the address phase and read/write data during the data phase. The **AS** (Address Strobe) pin demultiplexes the bus:

- **AS HIGH (address phase):** the internal address latch is transparent — AD0–AD7 passes through to the internal address register.
- **AS falling edge:** the latch freezes, capturing the register address present on AD0–AD7.
- **AS rising edge:** the latched address is **cleared**, regardless of CS state (per datasheet: *"The next rising edge that occurs on the AS bus clears the address regardless of whether CS is asserted"*).

On the ISA bus, address and data are already demultiplexed onto separate buses (A0–A19 and D0–D7). Since AD0–AD7 is connected to the ISA data bus (D0–D7 via the 74HCT245), the CPU data written to port 0x70 becomes the DS12885's internal register address. The ISA address bit A0 distinguishes the two ports:

| ISA Port | A0 | AS (RTCALE) | DS12885 phase | Operation |
|----------|----|---------|----|---|
| 0x70 | 0 | **pulsed HIGH during IOW#** | Address phase | CPU data → AD0–AD7 → latched as register address |
| 0x71 | 1 | **stays LOW** | Data phase | Read/write data at the previously latched register |

#### AS = RTCALE Implementation

The PLD generates `RTCALE = !AEN & !IOW & !A0 & address_decode` — the AS signal is fully qualified with the IOW# strobe, A0=0 (port 0x70), and the RTC address decode. This ensures the address latch opens **only** during the IOW# pulse of an `OUT 70h` instruction, preventing corruption from intervening bus cycles.

**Write to port 0x70 — Set RTC register address (Intel bus timing):**

```
  Signals during OUT 0x70, <reg_addr>:

  A0 ──────┐                              ┌─────
           └──────────────────────────────┘
  CS# ──┐                                ┌────
        └────────────────────────────────┘
  R/W ────────────┐             ┌─────────────
  (IOW#)          └─────────────┘
  AD0-7           ╔═══════reg_addr════════╗
  (via 74245)     ╚═══════════════════════╝
  AS ───────────────┐       ┌──────────────────
  (RTCALE)          └───────┘
                    ▲       ▲
                    │ IOW#  │ IOW# returns HIGH
                    │ goes  │ → AS falls
                    │ LOW   │ → latch freezes
                    │ → AS  │   reg_addr captured
                    │ HIGH  │
                    │ latch │
                    │ opens │
```

1. CPU places 0x70 on address bus → A0=0, PLD asserts BUF_EN# low (DS12885 CS# is permanently GND)
2. CPU drives register address on D0–D7 → through 74245 → AD0–7
3. IOW# goes LOW → PLD asserts RTCALE HIGH (address match + A0=0 + IOW# active) → AS=1 → address latch transparent
4. IOW# returns HIGH → RTCALE falls → AS falls → latch **freezes**, capturing the register address
5. Between bus cycles, RTCALE stays LOW — latch cannot be corrupted

**Read from port 0x71 — Read RTC register data:**

1. CPU places 0x71 on address bus → A0=1, BUF_EN# low, RTCALE=LOW
2. AS=0 → latch stays frozen with the previously set register address
3. IOR# goes LOW → PLD asserts RTCRD# LOW (address match + A0=1 + IOR#) → DS12885 RD# goes LOW → DS12885 drives read data onto AD0–AD7
4. 74245 direction = B→A (IOR#=0 → DIR=0) → data flows to ISA D0–D7
5. IOR# goes HIGH → RTCRD# goes HIGH → DS12885 releases bus, read complete

**Write to port 0x71 — Write RTC register data:**

1. Same addressing as read, but IOW# goes LOW → PLD asserts RTCWR# LOW (address match + A0=1 + IOW#) → DS12885 WR# goes LOW
2. IOW# goes HIGH → RTCWR# goes HIGH → WR# rising edge → DS12885 latches write data

#### Timing Compliance (from DS12885 AC specifications)

| Parameter | Symbol | Min | Our design meets? |
|-----------|--------|-----|-------------------|
| AS high pulse width | PWASH | 60 ns | Yes — AS=HIGH for IOW# pulse width (~350 ns on ISA) |
| Address valid before AS fall | tASL | 30 ns | Yes — data bus is stable before IOW# returns HIGH |
| Address hold after AS fall | tAHL | 10 ns | Yes — 74245 disable propagates through PLD (~15 ns), bus hold capacitance maintains data |
| AS fall to RD#/WR# assertion | tASED | 40 ns | Yes — AS falls at end of port 0x70 cycle; RD#/WR# asserts at start of port 0x71 cycle (≥200 ns apart at 4.77 MHz) |
| CS setup before RD# or WR# | tCS | 20 ns | Yes — CS# is permanently GND; always satisfied |

#### Address Latch Integrity Between Accesses

Since RTCALE is gated with IOW# and the full RTC address decode, the DS12885 address latch opens **only** during the IOW# pulse of an `OUT 70h` instruction. Between bus cycles, RTCALE stays LOW and the address latch remains frozen — no corruption is possible.

This is critical because the DS12885 datasheet states that AS-triggered address latching occurs regardless of CS state. A simpler approach like `AS = !A0` would toggle the latch during every bus cycle, re-latching whatever residual charge remains on the floating AD0-AD7 lines — an unreliable scheme that depends on bus capacitance retention.

### Oscillators

| Ref | Frequency | Purpose |
|-----|-----------|---------|
| Y1 | 32.768 kHz | RTC crystal |
| Y2 | 1.8432 MHz | UART clock oscillator |

### Data Bus Buffering

A 74HCT245 bidirectional transceiver decouples all peripheral data pins from the ISA bus:

- **ISA side** (A1–A8, pins 2–9) — connected to ISA D0–D7
- **Card side** (B1–B8, pins 11–18) — connected to RTC, UART, and PRN data pins; 8×10k pull-up resistor network to VCC ensures defined HIGH state when no device drives the bus (empty sockets read as 0xFF)
- **DIR** — connected to IOR# (IOR#=0 → B-to-A read; IOR#=1 → A-to-B write)
- **G#** — active-low enable driven by ATF22V10

### Address Decoding

A single ATF22V10 PLD decodes all port addresses and generates chip-select signals for every peripheral, plus the bus buffer enable and the RTC address strobe. No additional logic gates are required. Jumper inputs to the PLD allow per-device address and enable configuration.

See [plan.md](plan.md) for detailed PLD equations, pin assignments, and CUPL source.

### Jumper Configuration

#### Address Selection (3-pin headers: VCC / signal / GND)

| Jumper | Cap 1-2 (HIGH) | Cap 2-3 (LOW) |
|--------|----------------|---------------|
| JP_COM1_ADDR | 0x3F8 (COM1) | 0x2F8 (COM2) |
| JP_COM2_ADDR | 0x3E8 (COM3) | 0x2E8 (COM4) |
| JP_PRN_ADDR | 0x378 (LPT1) | 0x278 (LPT2) |

#### IRQ Selection (3-pin headers)

| Jumper | Position 1 | Position 2 |
|--------|-----------|-----------|
| JP_COM1_IRQ | IRQ4 | IRQ3 |
| JP_COM2_IRQ | IRQ4 | IRQ3 |
| JP_PRN_IRQ | IRQ7 | IRQ5 |
| JP_RTC_IRQ | IRQ2/9 / IRQ5 / IRQ6 / IRQ7 (1×5 header) | — |

#### Device Enable/Disable

Devices are enabled/disabled by installing or removing the chip from its DIP socket. A 10k×8 pull-up resistor network on the 74HCT245 B-side ensures that empty sockets read as 0xFF, which is the expected response for absent I/O devices. Software detection (e.g., BIOS UART scratch register test) correctly identifies the device as absent.

The RTC is always enabled (no disable mechanism).

## Manufacturing Notes

- **Board thickness:** 1.6 mm
- **Copper weight:** 1 oz (35 μm)
- **Surface finish:** HASL or ENIG
- **Solder mask / silkscreen:** any color; white silkscreen on top
- **Components:** through-hole
- **Form factor:** standard ISA 8-bit card dimensions
- **Connectors:** bracket-mounted DB-25 and DB-9; CR2032 battery holder

## Getting Started

### Required Software

- [KiCad](https://www.kicad.org/) 9.0 or later
- A PLD programmer (e.g. TL866II+) for the ATF22V10
- Git

### Opening the Project

1. Clone the repository
2. Open `ISA8_LPT_COM_RTC.kicad_pro` in KiCad
3. Schematic: `ISA8_LPT_COM_RTC.kicad_sch`
4. PCB layout: `ISA8_LPT_COM_RTC.kicad_pcb`

### Testing

1. Visually inspect for solder bridges; continuity-test power rails
2. Install card in an ISA slot
3. Verify I/O port addressing with DEBUG or a diagnostic utility
4. Test serial ports with a loopback plug or terminal program
5. Test parallel port with a printer or loopback connector
6. Verify RTC keeps time after power-off (battery backup)
7. Confirm interrupt operation for each device

## Troubleshooting

| Symptom | Check |
|---------|-------|
| Card not detected | ISA slot seating, power rails |
| Address conflicts | Reconfigure base address jumpers |
| IRQ conflicts | Change IRQ jumper settings |
| RTC loses time | Replace CR2032 battery |
| Serial errors | Baud rate, RS-232 cable/levels |
| Parallel port issues | Cable, port mode |

## Project Files

| File | Description |
|------|-------------|
| `ISA8_LPT_COM_RTC.kicad_pro` | KiCad project |
| `ISA8_LPT_COM_RTC.kicad_sch` | Schematic |
| `ISA8_LPT_COM_RTC.kicad_pcb` | PCB layout |
| `plan.md` | Implementation plan — PLD equations, pin maps, CUPL source |
| `pld/ISA8_DECODE.pld` | CUPL source for ATF22V10 |
| `pld/ISA8_DECODE.jed` | Compiled JEDEC fuse map |

## Datasheets

- [DS12885 / DS12C887A (RTC)](https://www.analog.com/media/en/technical-documentation/data-sheets/DS12885-DS12C887A.pdf)

## Version History

| Version | Notes |
|---------|-------|
| V2 | Current design iteration |
| V1 | Initial design (see parent directory) |

## Contributing

1. Document all schematic changes
2. Update PCB layout accordingly
3. Run DRC and ERC in KiCad
4. Update documentation as needed

## License

MIT
