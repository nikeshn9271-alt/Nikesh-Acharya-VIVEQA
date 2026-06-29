# board_b.xdc

**Constraints file for:** Board B (Backup / Heartbeat Master)
**Board:** Anmaya AT-STLN-ARTIX 7-001 (XC7A35T-FTG256-1)
**Clock:** 24 MHz MEMS oscillator (ASDM1, U4) on pin D13, Bank 15 MRCC

## Switch Map (all active LOW — switch ON = pin LOW = signal HIGH in Verilog via ~sw[x])
| Port | Pin | Function |
|------|-----|----------|
| sw[0] | C9 | Reset (active LOW) |
| sw[1] | B9 | Add mode (only active in failover) |
| sw[2] | G5 | Sub mode (only active in failover) |
| sw[3] | A7 | Enable transactions in failover |

## Button Map (btn[0]=digit 0 through btn[9]=digit 9)
| Port | Pin |
|------|-----|
| btn[0] | A13 |
| btn[1] | F5 |
| btn[2] | E3 |
| btn[3] | F2 |
| btn[4] | A12 |
| btn[5] | D6 |
| btn[6] | D3 |
| btn[7] | F3 |
| btn[8] | A5 |
| btn[9] | C6 |

## LED Map (one-hot heartbeat running light — always active)
| Port | Pin |
|------|-----|
| led[0] | D5 |
| led[1] | A3 |
| led[2] | B4 |
| led[3] | A4 |
| led[4] | E6 |
| led[5] | C13 |
| led[6] | C14 |
| led[7] | D14 |

## 7-Segment Display (MAX7219 SPI)
| Signal | Pin |
|--------|-----|
| seg_din | J15 |
| seg_clk | H12 |
| seg_cs | J16 |

## PMOD Pin Map
| Signal | Pin | Direction | Notes |
|--------|-----|-----------|-------|
| pmod_hb_pos[0] | T2 | Output | No pulldown on outputs |
| pmod_hb_pos[1] | R3 | Output | No pulldown on outputs |
| pmod_hb_pos[2] | T3 | Output | No pulldown on outputs |
| pmod_hb_tick | T4 | Output | No pulldown on outputs |
| pmod_sync_data | R1 | Output | No pulldown on outputs |
| pmod_sync_clk | N1 | Output | No pulldown on outputs |
| pmod_ack_valid | M1 | Input | PULLDOWN |
| pmod_bal_data | M2 | Input | PULLDOWN |
| pmod_bal_clk | P1 | Input | PULLDOWN |

> PULLDOWN on all inputs keeps lines LOW when Board A is disconnected or powered off, preventing false ACK or spurious serial frames from being detected.
> No PULLDOWN on output pins — pulldown on an output is illegal in XDC.

```xdc
set_property CFGBVS VCCO [current_design]
set_property CONFIG_VOLTAGE 3.3 [current_design]

create_clock -period 41.667 -name sys_clk [get_ports clk]
set_property -dict {PACKAGE_PIN D13 IOSTANDARD LVCMOS33} [get_ports clk]

set_property -dict {PACKAGE_PIN C9 IOSTANDARD LVCMOS33} [get_ports {sw[0]}]
set_property -dict {PACKAGE_PIN B9 IOSTANDARD LVCMOS33} [get_ports {sw[1]}]
set_property -dict {PACKAGE_PIN G5 IOSTANDARD LVCMOS33} [get_ports {sw[2]}]
set_property -dict {PACKAGE_PIN A7 IOSTANDARD LVCMOS33} [get_ports {sw[3]}]

set_property -dict {PACKAGE_PIN A13 IOSTANDARD LVCMOS33} [get_ports {btn[0]}]
set_property -dict {PACKAGE_PIN F5  IOSTANDARD LVCMOS33} [get_ports {btn[1]}]
set_property -dict {PACKAGE_PIN E3  IOSTANDARD LVCMOS33} [get_ports {btn[2]}]
set_property -dict {PACKAGE_PIN F2  IOSTANDARD LVCMOS33} [get_ports {btn[3]}]
set_property -dict {PACKAGE_PIN A12 IOSTANDARD LVCMOS33} [get_ports {btn[4]}]
set_property -dict {PACKAGE_PIN D6  IOSTANDARD LVCMOS33} [get_ports {btn[5]}]
set_property -dict {PACKAGE_PIN D3  IOSTANDARD LVCMOS33} [get_ports {btn[6]}]
set_property -dict {PACKAGE_PIN F3  IOSTANDARD LVCMOS33} [get_ports {btn[7]}]
set_property -dict {PACKAGE_PIN A5  IOSTANDARD LVCMOS33} [get_ports {btn[8]}]
set_property -dict {PACKAGE_PIN C6  IOSTANDARD LVCMOS33} [get_ports {btn[9]}]

set_property -dict {PACKAGE_PIN D5  IOSTANDARD LVCMOS33} [get_ports {led[0]}]
set_property -dict {PACKAGE_PIN A3  IOSTANDARD LVCMOS33} [get_ports {led[1]}]
set_property -dict {PACKAGE_PIN B4  IOSTANDARD LVCMOS33} [get_ports {led[2]}]
set_property -dict {PACKAGE_PIN A4  IOSTANDARD LVCMOS33} [get_ports {led[3]}]
set_property -dict {PACKAGE_PIN E6  IOSTANDARD LVCMOS33} [get_ports {led[4]}]
set_property -dict {PACKAGE_PIN C13 IOSTANDARD LVCMOS33} [get_ports {led[5]}]
set_property -dict {PACKAGE_PIN C14 IOSTANDARD LVCMOS33} [get_ports {led[6]}]
set_property -dict {PACKAGE_PIN D14 IOSTANDARD LVCMOS33} [get_ports {led[7]}]

set_property -dict {PACKAGE_PIN J15 IOSTANDARD LVCMOS33} [get_ports seg_din]
set_property -dict {PACKAGE_PIN H12 IOSTANDARD LVCMOS33} [get_ports seg_clk]
set_property -dict {PACKAGE_PIN J16 IOSTANDARD LVCMOS33} [get_ports seg_cs]

set_property -dict {PACKAGE_PIN T2 IOSTANDARD LVCMOS33} [get_ports {pmod_hb_pos[0]}]
set_property -dict {PACKAGE_PIN R3 IOSTANDARD LVCMOS33} [get_ports {pmod_hb_pos[1]}]
set_property -dict {PACKAGE_PIN T3 IOSTANDARD LVCMOS33} [get_ports {pmod_hb_pos[2]}]
set_property -dict {PACKAGE_PIN T4 IOSTANDARD LVCMOS33} [get_ports pmod_hb_tick]
set_property -dict {PACKAGE_PIN R1 IOSTANDARD LVCMOS33} [get_ports pmod_sync_data]
set_property -dict {PACKAGE_PIN N1 IOSTANDARD LVCMOS33} [get_ports pmod_sync_clk]

set_property -dict {PACKAGE_PIN M1 IOSTANDARD LVCMOS33} [get_ports pmod_ack_valid]
set_property -dict {PACKAGE_PIN M2 IOSTANDARD LVCMOS33} [get_ports pmod_bal_data]
set_property -dict {PACKAGE_PIN P1 IOSTANDARD LVCMOS33} [get_ports pmod_bal_clk]

set_property PULLDOWN true [get_ports pmod_ack_valid]
set_property PULLDOWN true [get_ports pmod_bal_data]
set_property PULLDOWN true [get_ports pmod_bal_clk]
```
