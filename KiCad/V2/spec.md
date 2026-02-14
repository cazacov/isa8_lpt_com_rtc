# ISA8 LPT COM RTC Board Specification

## Project Overview

This is a KiCad hardware design project for an 8-bit ISA expansion card that provides multiple legacy I/O interfaces and real-time clock functionality for retro computing systems.

## Features

- **PRN (Parallel Port)**: Standard printer/parallel port interface
- **COM (Serial Port)**: 2 x RS-232 serial communication interface
- **RTC (Real-Time Clock)**: Battery-backed real-time clock circuit
- **ISA 8-bit Interface**: Compatible with 8-bit ISA bus systems

## Target Systems

- IBM PC/XT and compatibles
- 8088/8086-based systems
- Retro computing builds

## Project Files

### KiCad Files
- `ISA8_LPT_COM_RTC.kicad_pro` - KiCad project file
- `ISA8_LPT_COM_RTC.kicad_sch` - Schematic design
- `ISA8_LPT_COM_RTC.kicad_pcb` - PCB layout
- `ISA8_LPT_COM_RTC.kicad_prl` - Project local settings

## Development Setup

### Required Software
- KiCad 9.0 or later
- Git for version control

### Opening the Project
1. Launch KiCad
2. Open `ISA8_LPT_COM_RTC.kicad_pro`
3. Access schematic editor for circuit design
4. Access PCB editor for board layout

## Design Specifications

### ISA Bus Interface
- 8-bit data bus (D0-D7)
- Address decoding for I/O port mapping
- Standard ISA edge connector pinout
- +5V and +12V/-12V power rails from bus

### Parallel Port (PRN)
- UM82C11-C printer adaper interface chip
- DB-25 connector
- Standard Centronics-compatible interface
- Configurable I/O base address 0x378 or 0x278
- Configurable interrupt IRQ5 or IRQ7
- CS signal is labeled PRN_CS (active low)

### 2 x Serial Port (UART)
- 16450/16550 UART compatible
- RS-232 voltage levels via GD75232N or compatible
- Serial Port 1
  - DB-9 connector 
  - Configurable I/O base address 0x3F8 or 0x2F8
  - Configurable interrupt IRQ3 or IRQ4
  - CS0, CS1 are connected to GND and always low
  - CS2 signal is labeled UART1_CS (active low)
- Serial port 2
  - 2x5 header pin connector
  - Configurable I/O base address 0x3E8 or 0x2E8
  - Configurable interrupt IRQ3 or IRQ4
  - CS0, CS1 are connected to GND and always low
  - CS2 signal is labeled UART2_CS (active low)

### Real-Time Clock (RTC)
- DS12885 RTC chip
- Battery backup CR2032
- CMOS RAM for BIOS settings
- Standard I/O ports 0x70/0x71
- Intel bus timing selected (MOT pin 1 not connected)

### Quartz Oscillators
- Y1 32768 Hz - Real time clock
- Y2 1.8432 MHz - UART chips

### Data bus buffering
Use 74245 bidirectional transceiver to decouples UART and PRN chips from the ISA bus.
ISA side (pins 2-9, A1-A8 on the 74245 datasheet) connected to the ISA bus pins D0-D7.
Card side (pins 11-18, B1-B8 on the datasheet) connected to data pins of RTC/COM1/COM2/LPT chips.
DIR pin of 74245 is connected to IOR# (because IOR#=0 on reads → DIR=0 = B→A, and IOR#=1 on writes → DIR=1 = A→B)

### Address decoding
Use ATF22V10 to decode port address and generate CS signals for PRN, UART (and maybe RTC) chips, G (enable) signal of 74245. Use additional logic gates if necessary.

### Jumpers
Jumpers allow to select:
- PRN chip
  - port 0x378 or 0x278
  - IRQ 5 or 7 
- UART1 chip
  - port 0x3F8 ot 0x2F8
  - IRQ 4 or 3
- UART2 chip
  - port 0x3E8 ot 0x2E8
  - IRQ 4 or 3
- RTC 
  - IRQ 5,6 or 7

Nice to have - be able to disable PRN and UART chips completelly with some jumper configuration

## Manufacturing Notes

### PCB Specifications
- Board thickness: 1.6mm standard
- Copper weight: 1oz (35μm)
- Surface finish: HASL or ENIG recommended
- Solder mask: Green standard (any color acceptable)
- Silkscreen: White on top side

### Assembly Considerations
- Through-hole components primarily
- Standard ISA card dimensions
- Bracket mounting for DB connectors
- 2032 Battery holder for RTC backup

## Testing Procedures

### Pre-Installation Checks
1. Visual inspection for solder bridges
2. Continuity testing on power rails
3. Verify no shorts between power and ground
4. Check ISA edge connector orientation

### Functional Testing
1. Install card in ISA slot
2. Test parallel port with printer or loopback
3. Test serial port with terminal or loopback
4. Verify RTC functionality and battery backup
5. Check interrupt operation
6. Verify I/O port addressing

## Configuration

### Jumper Settings
- I/O base address selection
- Interrupt selection
- Enable/disable individual Serial and LPT chips

### Software Configuration
- DOS/BIOS setup
- I/O port assignments
- Interrupt assignments
- Driver installation if required

## Compatibility Notes

- Standard 8-bit ISA bus timing
- Compatible with [8088 BIOS](https://github.com/skiselev/8088_bios)
- May require BIOS setup for I/O addresses
- DOS and early Windows compatible

## Troubleshooting

### Common Issues
- **No detection**: Check ISA slot seating, verify power
- **Address conflicts**: Reconfigure I/O base addresses
- **Interrupt conflicts**: Change IRQ settings
- **RTC not keeping time**: Replace backup battery
- **Serial errors**: Check baud rate, verify RS-232 levels
- **Parallel port issues**: Check cable, verify port mode

## Development Status

**Current Version**: V2

### Revision History
- **V2**: Current design iteration
- Previous versions maintained in parent directories

## Contributing

When making changes to this design:
1. Document all schematic changes
2. Update PCB layout accordingly
3. Run Design Rule Check (DRC)
4. Run Electrical Rule Check (ERC)
5. Update this documentation

## License

MIT

## Datasheets
- [DS12885](https://www.analog.com/media/en/technical-documentation/data-sheets/DS12885-DS12C887A.pdf)
