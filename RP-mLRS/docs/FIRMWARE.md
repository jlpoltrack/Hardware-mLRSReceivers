# Firmware notes

What the firmware has to do because of how the boards are wired. Pin numbers are RP2354A GPIOs.

## GPIO map

| Function | GPIO | Notes |
|----------|------|-------|
| BOOT button (via R5) | 0 | input, low = pressed; never an output |
| **CANFD build:** CAN INT | 1 | |
| **CANFD build:** CAN SCK / MOSI / MISO / CS | 2 / 3 / 4 / 5 | SPI0 |
| **CAN build:** CAN RX / TX | 1 / 4 | can2040 (PIO); GPIO2, 3, 5 unused |
| LR2021 IRQ / RESET | 6 / 7 | |
| LR2021 MISO | 8 | SPI1 RX |
| LR2021 MOSI | 11 | SPI1 TX |
| LR2021 BUSY / NSS / SCK | 12 / 13 / 14 | SPI1 |
| LED green / red | 19 / 29 | |
| UART1 TX / RX | 20 / 21 | TX / RX pads |
| UART0 TX | 28 | TX1 pad |

## Core regulator

VREG_FB is tied to GND because the core runs from the external 1.1 V LDO (U2, TLV74311; RP2350 datasheet §6.3.7).
Set the VREG `HIZ` bit at start-up to turn the internal regulator off.

## Clock

12 MHz 3.3 V CMOS oscillator (Y1) on XIN, XOUT open.

## CANFD build (U5 fitted): MCP251863

References (Microchip):

- [MCP251863 data sheet, DS20006624B](https://ww1.microchip.com/downloads/aemDocuments/documents/APID/ProductDocuments/DataSheets/MCP251863-External-CAN-FD-Controller-with-Integrated-Transceiver-DS20006624.pdf)
- [MCP25xxFD Family Reference Manual (FRM), DS20005678E](https://ww1.microchip.com/downloads/aemDocuments/documents/OTH/ProductDocuments/ReferenceManuals/MCP25XXFD-CAN-FD-Controller-Module-Family-Reference-Manual-DS20005678E.pdf)
- [MCP251863 errata, DS80000984B](https://ww1.microchip.com/downloads/aemDocuments/documents/APID/ProductDocuments/Errata/MCP251863-Silicon-Errata-and-Data-Sheet-Clarification-DS80000984.pdf)

- **#1 SPI reads can return wrong data.** Use `READ_CRC` for status/control registers (CiCON, CiINT, CiTXIF, CiRXIF, CiTBC, CiTXREQ, CiTREC, CiBDIAG0/1, CiTXQSTA, CiFIFOSTAm, …) and retry on a CRC mismatch. Read an RX FIFO only after its flag says a message is loaded. Avoid FIFOs 7, 15, 23 and 31.
- **#2 ECC.** Single-error correction is unreliable. Enable `SECIE` and `DEDIE`, and treat both interrupts as "the RAM word at ERRADDR is corrupt".
- **#3 SFR address rollover doesn't work** (0x3FF → 0x400, 0xFFF → 0x000). Don't do burst reads across those boundaries.
- **#6 FIFOCI** in an RX FIFOSTA can read wrong, with a matching CRC. Read it more than once and reject values that don't make sense.
- **CRC register clarification:** `FERRIF` can set after a bare CS pulse (CS low then high with no clocks) if the last transfer was READ_CRC, WRITE_CRC or WRITE_SAFE. Clear it in the CRC register.

## CAN build (U7 fitted)

There is no CAN controller. can2040 (PIO) runs on GPIO1 (RX) and GPIO4 (TX). The SN65HVD230's Rs pin is wired for full speed, so it has no standby to manage. Only the core-regulator note above applies.
