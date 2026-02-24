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
- IRQ polarity inversion: DS12885 IRQ output is active-low open-drain; an NPN transistor (Q1, 2N3904) with a 330Ω series current-limiting resistor converts it to the active-high signal expected by the ISA bus 8259A PIC (see [design.md](design.md) Section 6 for circuit details)
- On PC/XT (single 8259A), the IRQ2 jumper position triggers IRQ2 (INT 0Ah); on AT-class systems (cascaded 8259A), the same physical IRQ2 line is routed to the slave PIC as IRQ9 (INT 71h), which the BIOS redirects to INT 0Ah for compatibility
- Multiplexed address/data bus: CPU writes register address to port 0x70, then reads/writes data at port 0x71 (see [design.md](design.md) Section 5 for interface details, timing diagrams, and PLD equations)

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

See [design.md](design.md) for detailed PLD equations, pin assignments, and CUPL source.

### Jumper Configuration

#### Address Selection (3-pin headers: VCC / signal / GND)

| Jumper | Cap 1-2 (HIGH) | Cap 2-3 (LOW) |
|--------|----------------|---------------|
| J1 (COM1 Addr) | 0x3F8 (COM1) | 0x2F8 (COM2) |
| J3 (COM2 Addr) | 0x3E8 (COM3) | 0x2E8 (COM4) |
| J5 (PRN Addr) | 0x378 (LPT1) | 0x278 (LPT2) |

#### IRQ Selection (3-pin headers)

| Jumper | Position 1 | Position 2 |
|--------|-----------|-----------|
| J2 (COM1 IRQ) | IRQ4 | IRQ3 |
| J4 (COM2 IRQ) | IRQ4 | IRQ3 |
| J6 (PRN IRQ) | IRQ7 | IRQ5 |
| J7 (RTC IRQ) | IRQ2/9 / IRQ5 / IRQ6 / IRQ7 (1×5 header) | — |

#### RTC CMOS Clear

J8 is a 2-pin header connected between DS12885 RCLR# (pin 21) and GND. The RCLR# pin has an internal pull-up inside the DS12885. Momentarily shorting J8 with a jumper cap (while the system is powered off) clears all CMOS RAM and RTC registers to defaults.

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
- **Connectors:** bracket-mounted DB-25 and DE-9; CR2032 battery holder

## Getting Started

### Required Software

- [KiCad](https://www.kicad.org/) 9.0 or later
- A PLD programmer (e.g. TL866II+) for the ATF22V10
- Git

### Opening the Project

1. Clone the repository
2. Open `KiCad/V2/ISA8_LPT_COM_RTC.kicad_pro` in KiCad
3. Schematic: `KiCad/V2/ISA8_LPT_COM_RTC.kicad_sch`
4. PCB layout: `KiCad/V2/ISA8_LPT_COM_RTC.kicad_pcb`

### Testing

1. Visually inspect for solder bridges; continuity-test power rails
2. Install card in an ISA slot
3. Verify I/O port addressing with DEBUG or a diagnostic utility
4. Test serial ports with a loopback plug or terminal program
5. Test parallel port with a printer or loopback connector
6. Verify RTC keeps time after power-off (battery backup)
7. Confirm interrupt operation for each device

## KiCad

Third party libraries should be stored in locations referenced with environment variables:
- **KICADLIB_SKISELEV** [Sergey Kiselev's library](https://github.com/skiselev/my_kicad_library)
- **KICADLIB_VICTOR** [Victor's (my) library](https://github.com/cazacov/VictorLib)

## Troubleshooting

| Symptom | Check |
|---------|-------|
| Card not detected | ISA slot seating, power rails |
| Address conflicts | Reconfigure base address jumpers, ensure ATF22V10 firmware is programmed properly |
| IRQ conflicts | Change IRQ jumper settings |
| RTC loses time | Replace CR2032 battery |
| Serial errors | Baud rate, RS-232 cable/levels |
| Parallel port issues | Cable, port mode |

## Project Files

| File | Description |
|------|-------------|
| `KiCad/V2/ISA8_LPT_COM_RTC.kicad_pro` | KiCad project |
| `KiCad/V2/ISA8_LPT_COM_RTC.kicad_sch` | Schematic |
| `KiCad/V2/ISA8_LPT_COM_RTC.kicad_pcb` | PCB layout |
| `design.md` | Implementation plan — PLD equations, pin maps, CUPL source |
| `pld/ISA8_DECODE.pld` | CUPL source for ATF22V10 |
| `pld/ISA8_DECODE.jed` | Compiled JEDEC fuse map |

## Datasheets

- [DS12885 / DS12C887A (RTC)](https://www.analog.com/media/en/technical-documentation/data-sheets/DS12885-DS12C887A.pdf)

## Version History

| Version | Notes |
|---------|-------|
| V2 | Current design iteration |
| V1 | Initial design (see `KiCad/V1/`) |

## Contributing

1. Document all schematic changes
2. Update PCB layout accordingly
3. Run DRC and ERC in KiCad
4. Update documentation as needed

## Credits

- [Sergey Kiselev's projects](https://github.com/skiselev)
- [Aitor Gómez García's RTC8088 project](https://github.com/spark2k06/RTC8088)

## License

MIT