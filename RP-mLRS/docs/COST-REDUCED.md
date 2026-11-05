# RP-mLRS – cost-reduced variant (notes)

As of 2026-10-17, from the JLC parts quote for 5 boards (Rev G, top side, Economic PCBA).

V1.1 (2026-10-24) since: Y1 is a 12 MHz oscillator (C5280582, C10 / R11 gone) and J2 / J3 are CJT A1257WR-S-4P
(C225127, $0.12 vs the JST's $0.31): about $0.50 per board less at 5 boards. The tables below are V1.0's.

**Status:** Option A is the CAN build of [`RP-mLRS-DUAL/`](../RP-mLRS-DUAL) (can2040 on GPIO1 RX / GPIO4 TX + SN65HVD230).

The assembled parts cost **$33.84 for 5 boards (about $6.77 per board)**, and the MCP251863 CAN controller alone
is 47 % of it. The RP2354A is the only other large item, and it stays. So the one real lever is CAN.

## Where the parts cost goes

| Part | 5 boards | Per board | Share | Lib type |
|---|---|---|---|---|
| U5 MCP251863 | $16.02 | $3.20 | 47 % | Extended |
| U3 RP2354A | $6.31 | $1.26 | 19 % | Extended |
| Y1 12 MHz crystal (ABM8-272-T3) | $3.24 | $0.65 | 10 % | Extended |
| J2, J3 JST-GH (2 per board) | $3.05 | $0.61 | 9 % | Extended |
| U1 AMS1117-3.3 | $1.04 | $0.21 | 3 % | Basic |
| J1 USB-C | $0.93 | $0.19 | 3 % | Extended |
| U2 TLV74311 | $0.84 | $0.17 | 2 % | Extended |
| Everything else (passives, LEDs, diode, button, Y2) | $2.41 | $0.48 | 7 % | mostly Basic |

Two costs sit outside this quote:

- **LR2021 module**: bought from NiceRF and hand-soldered; likely the most expensive single part.
- **JLC assembly fees**: about $40 for 5 boards, more than all the parts. Each extended part type adds a $3 loading
  fee per order (Economic PCBA), so at small quantities removing a part type can save more than the part itself.

## Option A: keep CAN, drop the MCP251863

Run classic CAN in software on one of the RP2350's PIO blocks (the can2040 library) and add a small 3.3 V
transceiver. This saves about **$2.75 per board and $6 per order** in loading fees.

| Change | Per board | Per order |
|---|---|---|
| Remove U5 MCP251863 | −$3.20 | −$3 loading fee |
| Remove Y2 40 MHz crystal + C11/C12 | −$0.13 | −$3 loading fee |
| Add SN65HVD230 transceiver (SOIC-8, C12084, as on SimpleCANBreakout) | +$0.57 | none (no loading fee) |

- **Board:** an 8-pin transceiver replaces the 28-pin MCP251863 and its crystal, so the CAN corner gets simpler.
  CAN TX/RX move to two free GPIOs; the SPI0 lines and the CAN interrupt are freed.
- **Lost:** CAN FD. DroneCAN normally runs on classic CAN at 1 Mbit/s, so this is unlikely to matter.
- **Firmware:** mLRS already supports can2040, so no new driver is needed. can2040 takes one PIO block and some
  interrupt time on one core; the RP2350 has three PIO blocks, and the radio uses hardware SPI1, not PIO.

## Option B: UART-only, no CAN

Remove the whole CAN section. This saves about **$4 per board and $9 per order** (three extended types: MCP251863,
Y2, JST-GH).

- **Removed:** U5, Y2, C11/C12, R6 + JP1 (termination), J2/J3, and U5's decoupling caps.
- **Board:** the bottom-left quarter (U5 + connectors) frees up, so the board could shrink.
- **Lost:** DroneCAN and the CAN connectors; telemetry/control only over the UART pads (TX, RX, TX1) and USB.

## Smaller savings (only if CAN stays)

| Change | Saves | Trade-off |
|---|---|---|
| One JST-GH instead of two | $0.30 per board | No daisy-chaining the bus through the board |
| AMS1117 → XC6206 (C5446, basic) | $0.06 per board ($0.1442 vs $0.2085) | 200 mA: a shorted CAN bus (up to 250 mA from the transceiver) would sag 3.3 V and reset the board; not worth it |

## What not to change

- **Y1 crystal (ABM8-272-T3, V1.0):** Raspberry Pi's recommended part for the RP2350; its oscillator wants low ESR
  (about 50 Ω) and about 10 pF load. The obvious cheap 12 MHz part, C9002, is 80 Ω / 20 pF, so start-up could be
  unreliable — a real risk to save $0.55 per board. Replaced in V1.1 by a 12 MHz CMOS oscillator (C5280582): no start-up
  risk, cheaper than the ABM8.
- **RP2354A:** the core of the board.
- **TLV74311 (1.1 V core LDO):** the alternative is the RP2350's internal switching regulator, which needs an
  inductor and careful layout to save about $0.17.
- **Everything else** is cents per part.

## Recommendation and open questions

Go with **Option A**: replace the MCP251863 with can2040 + an SN65HVD230. mLRS already supports can2040, so DroneCAN
stays with no firmware work, and a 5-board order gets about **$20 cheaper**.

| 5-board order | Rev G | Option A | Saved |
|---|---|---|---|
| Parts | $33.84 | about $20.05 | $13.80 |
| Extended-part loading fees | $21 (7 types) | $15 (5 types) | $6 |
| **Total** | **$54.84** | **about $35.05** | **about $19.80 (36 %)** |

Assembly setup, stencil, PCB and the LR2021 module are the same either way.

- **Smaller savings:** keep both JST-GH (daisy-chaining is worth $0.30) and keep the AMS1117: the XC6206 saves only
  $0.06 per board and can't supply a shorted CAN bus (the SN65HVD230 can draw up to 250 mA; normal worst case on
  3.3 V is about 145 mA).
- **Option B** (UART-only) is only worth it if a smaller board without CAN is wanted for its own sake.

- [x] mLRS supports can2040 (PIO CAN), so the MCP251863 driver isn't needed.
- [x] SN65HVD230 (C12084): $0.57 each, no loading fee.
- [x] CAN GPIOs: GPIO1 = RX, GPIO4 = TX (they reuse the old INT and MISO escape lanes; GPIO2, 3, 5 unused).
- [x] Option A laid out as a separate board, RP-mLRS-CAN (not a revision of RP-mLRS-CANFD).

Sources: JLC quote screenshot (5 boards, 2026-10-17); JLC part pages for
[C9002](https://jlcpcb.com/partdetail/C9002), [C20625731](https://jlcpcb.com/partdetail/C20625731),
[C5226885](https://jlcpcb.com/partdetail/C5226885), [C41378174](https://jlcpcb.com/partdetail/C41378174);
assembly fees: [JLCPCB PCB Assembly FAQs](https://jlcpcb.com/help/article/pcb-assembly-faqs).
