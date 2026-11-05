# F1-CAN-Receiver design spec

mLRS 2.4 GHz receiver with CAN: STM32F103C8T6, two E28-2G4M27S (SX1281 + PA/LNA, diversity), classic CAN to
the flight controller. Based on Olli's dual-E28 board02 (`../mLRS-hardware/olliw-stm32-based/rx-tx-e28dual-board02-f103cb`),
firmware target `rx-diy-e28dual-board02-f103cb`. 4 layers (In1 GND, In2 +3V3 with a +5V island), single-sided assembly, 50 x 30 mm (24 mm
plus 3 mm top and bottom for hand-soldering the E28 pads): the E28s' IPEX ends
overhang the left edge (21 mm of each module on the board).

## Decisions
- No USB. PA9/PA10 (USART1) are the serial port and the F1 ROM bootloader port: first flash via a USB-UART adapter
  (hold BOOT0, power up), then mLRS updates the same way or over SWD.
- CAN on PA11/PA12: the mLRS F1 DroneCAN driver hardcodes these pins (`modules/stm32-dronecan-lib/stdstm32-can.h`).
- Two E28s on board02's pins, DNP (hand-soldered after assembly). Everything is on the top side except M2 and its
  two 0805 decaps (C18, C19, DNP); SX2 can be left unfitted for a single-radio build. Antennas via the modules' IPEX.
- Two plain UARTs on pads: USART1 (serial + bootloader) and USART2 (board02's OUT port, SBus/CRSF). No XOR
  inverters: board02's firmware-selectable inversion is dropped, so an inverted SBus needs an external inverter.
- 16 MHz crystal (C13738, basic, 3225): much smaller than the 8 MHz 5032 part.
- Power from the CAN 5 V into an AMS1117-3.3.
- Design blocks: `ldo-ams1117-3v3` (U2, C1/C5 in, C2/C3 out), `can-sn65hvd230` (U3, C4, C6),
  `can-jstgh-pair-120` (J1/J2, R1 + JP1, C20/C21 10u on +5V), `status-leds` (LED1/LED2, R4/R8 on the cathodes);
  the last two keep the blocks' circuits but are placed by hand to fit 50 mm.
  `stm32f103-core` (U1, Y1 + C7/C8, SW1 + R2, R3, C9, C10-C15) was cut from this board.

## Pin map
Peripherals stay on board02's pins (no F1 remap helps: the bootloader needs USART1 on PA9/PA10, the mLRS CAN driver
PA11/PA12; SPI1's remap keeps the same SCK/MISO/MOSI order on the far side). The E28 bodies reach to x 121.1, 1 mm from
U1's left pins, so the modules' pads are entered from under the modules (board02 does the same): M1's nets on F.Cu,
straight left out of U1's left pins in M1's pad order; M2's nets through vias under U1, then B.Cu under M2. SPI1_SCK
hops over MISO/MOSI (the module wants MISO, MOSI, SCK top to bottom); SX1_BUSY / SX1_DIO1 come round on B.Cu.
The firmware needs its own hal (pins + 16 MHz HSE); board02's pin defines no longer match.

| Pin | Net | Route |
|---|---|---|
| PA0 | SX2_DIO1 | EXTI0; via under U1, B.Cu |
| PA1 | SX2_BUSY | via under U1, B.Cu |
| PA2 / PA3 | UART2_TX / RX | USART2, OUT pads |
| PA4 | DBG_TX | mLRS software UART (TX only, needs a timer) |
| PA5 / PA6 / PA7 | SPI1 SCK / MISO / MOSI | SX1 (M1, F.Cu) |
| PA8 | LED_GREEN | |
| PA9 / PA10 | USART1 TX / RX | **serial + bootloader** |
| PA11 / PA12 | CAN_RX / CAN_TX | to transceiver |
| PA13 / PA14 | SWDIO / SWCLK | SWD pads (JTAG off: PA15, PB3, PB4 are GPIOs) |
| PA15 | SX1_DIO1 | EXTI15 |
| PB0 | SPI1_CS | SX1 |
| PB1 | SX1_RX_EN | |
| PB2 | BOOT1 | 10k to GND |
| PB3 | SX1_BUSY | |
| PB4 | BUTTON | bind button to GND |
| PB5 | - | unused |
| PB6 | SPI2_CS | SX2 |
| PB7 | SX2_RX_EN | |
| PB8 | SX2_TX_EN | |
| PB9 | SX2_RESET | |
| PB10 | SX1_TX_EN | |
| PB11 | SX1_RESET | |
| PB12 | LED_RED | |
| PB13 / PB14 / PB15 | SPI2 SCK / MISO / MOSI | SX2 (M2, B.Cu) |
| PC13 / PC14 / PC15 | - | unused (no LED on them: 3 mA sink only, never a source) |
| PD0 / PD1 | 16 MHz crystal | **board02 runs an 8 MHz HSE: the hal needs HSE/2 into the PLL** |
| BOOT0 | | 10k to GND + button to 3V3 |
| NRST | | 100 nF to GND (no pad) |

## Connectors
- **J1, J2 CAN**: JST-GH-compatible 4-pin (CJT A1257WR-S-4P since V1.1; JST SM04B-GHS-TB was out of stock), order 1 5V, 2 CAN_H, 3 CAN_L, 4 GND; in parallel for daisy-chaining.
  120 Ω termination through a solder jumper (closed = terminated).
All pads are 2 mm square SMD (`TestPoint_Pad_2.0x2.0mm`), all on the back, labelled there.
- Pads: UART1 + UART2 in one line on B.Cu under J1/J2, SWD on B.Cu under U1, debug on B.Cu top left (wire
  pads never in two rows facing the same way).
- **UART1** (USART1) TP1-TP4: 5V, TX, RX, GND, so a USB-UART adapter powers the board while flashing.
- **UART2** (USART2, board02's OUT port) TP5-TP7: TX, RX, GND (no 5V).
- **SWD** TP8-TP11: GND, SWCLK, SWDIO, 3V3 read from the back. NRST has no
  pad. No reset button: hold BOOT0 and power up.
- **Debug** (software UART on PA4, TX only) TP12-TP13: GND, DBG.

## BOM (LCSC)

| Ref | Part | LCSC | JLC | Note |
|---|---|---|---|---|
| U1 | STM32F103C8T6 | C8734 | preferred | |
| U2 | AMS1117-3.3 | C6186 | basic | |
| U3 | SN65HVD230DR | C12084 | preferred | Rs (pin 8) to GND = high speed |
| M1, M2 | E28-2G4M27S | C411312 | DNP | hand-soldered; LCSC number kept for reference, field DNP |
| Y1 | 16 MHz crystal CL 9 pF (X322516MLB4SI, 3225) | C13738 | basic | + 2x 10 pF C32949 |
| J1, J2 | CJT A1257WR-S-4P | C225127 | extended | `FT Position Offset = 0,0.3` (JLC's model sits 0.3 forward) |
| SW1, SW2 | TS-1088 | C720477 | basic | BOOT0, BIND |
| LED1/2 | red C84256, green C2297 | | basic | 330 Ω C25104 each |
| R | 10k C25744 (BOOT0, BOOT1), 120 Ω C25079 | | basic | |
| C | 100n C307331 (0402 50 V), 10u C96446 (0603 25 V), 1u C52923, 10p C32949 | | basic | |
| C18, C19 | 10u C15850, 100n C49678, 0805 hand-solder pads | | DNP | M2's decaps, bottom side |

One extended part type: the GH connectors (about $3 in fees). E28s are DNP, so they are outside the JLC order.
JLC quote (2026-10-24, V1.1): 5 assembled boards $34.76, under the $59 line.

## Firmware (new hal, copy of `rx-hal-diy-e28dual-board02-f103cb.h`)
- `UARTB_USE_UART1_PA9PA10` (serial), debug as a software UART on PA4 (PB10/PB11 are SX1 TX_EN / RESET here),
  UART2 for OUT unchanged; drop the XOR inversion-select pins (PB9, PC15).
- HSE 16 MHz: set the PLL source to HSE/2 (`RCC_HSE_PREDIV_DIV2`) so SYSCLK stays 72 MHz.
- `#define DEVICE_HAS_DRONECAN`, enable `HAL_CAN_MODULE_ENABLED` in `stm32f1xx_hal_conf.h`.
- Check that flash and RAM fit on the F103 (20 KB RAM, `EE_START_PAGE 124` assumes 128 KB flash) **before ordering**.

## Notes
- The LDO is the hot part: about 1 W while one E28 transmits at 27 dBm (580 mA from 5 V). Give it a copper pour.
- All 5 V inputs share one net with no diodes, so a USB-UART adapter plugged in on the bench back-feeds the CAN 5 V.
- PB3, PB4 and PA15 are JTAG pins. The firmware already switches to SWD only, as on board02.
