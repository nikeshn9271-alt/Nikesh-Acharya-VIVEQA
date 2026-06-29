# Fault-Tolerant Banking System
### Team: Niranjan & Nikesh
### Platform: Anmaya AT-STLN-ARTIX 7-001 (Xilinx XC7A35T) | Clock: 24 MHz

---

## 1. Project Overview

The Fault-Tolerant Banking System is a hardware implementation of a fault-tolerant financial transaction system using two physical FPGA boards. The system models a real-world banking architecture where a primary processing unit handles all customer transactions while a secondary backup unit continuously monitors the primary and stores a live copy of the account balance.

If the primary unit fails — whether due to a power outage, hardware fault, or maintenance — the backup unit automatically detects the failure within 3 seconds and seamlessly takes over all banking operations without any data loss. When the primary unit is restored, it receives the latest balance from the backup and resumes normal operation.

The system supports:
- Deposits (addition) and withdrawals (subtraction) with a balance range of 0 to 9999
- Automatic failover and recovery with zero data loss
- Live balance synchronization between both units via a dedicated serial communication link
- Fault simulation controls for demonstration purposes

---

## 2. Design and Architecture

### 2.1 System Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                    PMOD INTERFACE (9 wires)                  │
│                                                              │
│  Board B (Backup)          ←→          Board A (Primary)     │
│  ┌──────────────┐                      ┌──────────────┐      │
│  │ Heartbeat    │──hb_pos[2:0]────────▶│ Heartbeat    │     │
│  │ Generator    │──hb_tick────────────▶│ Receiver     │     │
│  │              │◀─ack_valid───────────│              │     │
│  │ Balance      │◀─bal_data────────────│ Balance      │     │
│  │ Receiver     │◀─bal_clk─────────────│ Sender       │     │
│  │              │──sync_data──────────▶│              │     │
│  │ Sync Sender  │──sync_clk───────────▶│ Sync Receiver│     │
│  └──────────────┘                      └──────────────┘      │
│  • Monitors ACK timeout                • All transactions    │
│  • Takes over on failure               • Sends balance to B  │
│  • Sends balance to A on recovery      • Loads balance from B│
└──────────────────────────────────────────────────────────────┘
```

### 2.2 PMOD Pin Mapping

| Pin       | Signal    | Direction | Purpose                    |
|-----------|-----------|-----------|----------------------------|
| IO_0 (T2) | hb_pos[0] | B → A     | Heartbeat position bit 0   |
| IO_1 (R3) | hb_pos[1] | B → A     | Heartbeat position bit 1   |
| IO_2 (T3) | hb_pos[2] | B → A     | Heartbeat position bit 2   |
| IO_3 (T4) | hb_tick   | B → A     | Heartbeat tick pulse       |
| IO_4 (M1) | ack_valid | A → B     | Board A alive signal       |
| IO_5 (M2) | bal_data  | A → B     | Serial balance data A to B |
| IO_6 (P1) | bal_clk   | A → B     | Serial clock A to B        |
| IO_7 (R1) | sync_data | B → A     | Serial balance data B to A |
| IO_8 (N1) | sync_clk  | B → A     | Serial clock B to A        |

### 2.3 Operational Roles

**Board A — Primary Unit:**
- Handles all user transactions (deposits and withdrawals)
- Displays current balance on 7-segment display
- Follows heartbeat from Board B and sends ACK
- Continuously transmits balance to Board B via serial link
- Receives balance from Board B after freeze or shutdown is released

**Board B — Backup Unit:**
- Generates heartbeat signal every 500ms
- Monitors ACK from Board A with 3 second timeout
- Silently stores incoming balance from Board A
- Activates as primary after confirmed Board A failure
- Sends current balance to Board A on reconnection

---

## 3. Implementation Approach

### 3.1 Heartbeat Protocol

Board B generates a 3-bit heartbeat position counter that increments every 500ms, driving a single LED across 8 LEDs (running light pattern). This position is broadcast to Board A via PMOD. Board A follows this position on its own LEDs and sends back an acknowledgement pulse (`ack_valid`) every time the position changes.

Board B monitors the ACK using a 4-bit shift filter. Only when 4 consecutive HIGH samples are received does it confirm Board A is alive and reset the timeout counter. This prevents false alarms from electrical noise on the PMOD wire.

### 3.2 Failover Detection

Board B maintains a timeout counter that counts up whenever the ACK filter does not reach `4'b1111`. If no valid ACK is received for 3 seconds (72,000,000 clock cycles at 24MHz), `board_a_alive` is declared LOW and `board_b_active` goes HIGH.

Board B activates as primary whenever Board A is absent — whether Board A was previously connected or not. This ensures Board B is always ready to handle transactions independently if needed.

### 3.3 Balance Synchronization — A to B

Board A continuously transmits its 14-bit balance to Board B using a custom framed serial protocol:
- A sync pulse (data HIGH during clock HIGH) marks the start of each frame
- 14 data bits follow MSB first
- Board B detects the sync pulse and receives the frame
- On completion Board B updates `received_balance`
- Board B's ledger is updated from `received_balance` every time a complete frame arrives while Board B is in standby mode

### 3.4 Balance Synchronization — B to A

Board B sends its stored balance back to Board A in two situations:
1. When Board A reconnects after a failure (`board_a_returned` pulse)
2. Periodically every 1 second to ensure Board A always has the latest value

Board A loads this balance into its ledger only when `wait_sync` is active. This flag is set when freeze or shutdown is released and cleared once a valid sync frame is received. This prevents Board B's periodic sends from overwriting Board A's balance during normal operation.

### 3.5 Transaction Processing

Users enter a 4-digit amount using the keypad (keys 0-9). The digit entry FSM collects digits one at a time and fires `value_ready` only after the 4th digit is entered. The ledger core then adds or subtracts the entered value from the current balance with overflow and underflow protection (maximum 9999, minimum 0).

### 3.6 Freeze and Shutdown Simulation

- **Freeze switch:** Halts heartbeat LED, stops ACK to Board B, blocks transactions, shows "Err" on display. Board B detects ACK loss after 3 seconds and takes over. When freeze is released, Board A waits for sync from Board B before resuming.
- **Shutdown switch:** Simulates complete power failure. LEDs turn off, display blanks, ACK stops, balance sender stops. Board B takes over after 3 seconds. When shutdown is released, Board A waits for balance sync from Board B.

---

## 4. Module Descriptions

### 4.1 `board_a_top.v` — Primary Board Top Module
Integrates all Board A logic. Manages heartbeat following, ACK generation, balance serial sender and receiver, digit entry, ledger, and display. Controls freeze and shutdown behavior including `wait_sync` state management.

| Port | Direction | Description |
|------|-----------|-------------|
| clk | Input | 24 MHz system clock |
| sw[4:0] | Input | sw[0]=add, sw[1]=sub, sw[2]=freeze, sw[3]=shutdown, sw[4]=enable |
| btn[15:0] | Input | Keypad buttons — btn[0]-btn[9] are digits |
| led[7:0] | Output | Heartbeat running light (8 LEDs) |
| seg_din/clk/cs | Output | MAX7219 SPI for 7-segment display |
| pmod_ack_valid | Output | Heartbeat acknowledgement to Board B |
| pmod_bal_data/clk | Output | Serial balance transmission to Board B |
| pmod_hb_pos[2:0] | Input | Heartbeat position from Board B |
| pmod_hb_tick | Input | Heartbeat tick pulse from Board B |
| pmod_sync_data/clk | Input | Balance sync from Board B |

### 4.2 `board_b_top.v` — Backup Board Top Module
Integrates all Board B logic. Generates heartbeat, monitors ACK with timeout and noise filtering, receives balance from Board A, sends balance to Board A, manages failover state, and controls transaction capability gating.

| Port | Direction | Description |
|------|-----------|-------------|
| clk | Input | 24 MHz system clock |
| sw[3:0] | Input | sw[0]=rst, sw[1]=add, sw[2]=sub, sw[3]=enable |
| btn[15:0] | Input | Keypad buttons (active only during failover) |
| led[7:0] | Output | Heartbeat running light (always active) |
| seg_din/clk/cs | Output | MAX7219 SPI (active only during failover) |
| pmod_hb_pos[2:0] | Output | Heartbeat position to Board A |
| pmod_hb_tick | Output | Heartbeat tick to Board A |
| pmod_ack_valid | Input | ACK from Board A |
| pmod_bal_data/clk | Input | Balance from Board A |
| pmod_sync_data/clk | Output | Balance sync to Board A |

### 4.3 `debounce.v` — Button Debouncer (Shared)
Eliminates button bounce using a 20-bit counter (~42ms at 24MHz). Outputs a single 1-cycle pulse on the rising edge of a stable button press. Used for all 10 keypad digit buttons on both boards.

### 4.4 `digit_entry_fsm.v` — Digit Entry FSM (Shared)
Collects 4 keypad button presses one at a time and assembles them into a 14-bit integer value. Fires `value_ready` for exactly one clock cycle after the 4th digit is entered. Resets automatically after transaction completion or when mode switches are turned off.

### 4.5 `ledger_core.v` — Balance Manager (Shared)
Maintains the account balance. Supports addition and subtraction with bounds checking (0 to 9999). Uses a 15-bit intermediate wire for addition to prevent 14-bit overflow wrapping. Accepts `sync_in` to load an externally provided balance value.

| Parameter | Value |
|-----------|-------|
| Initial balance | 1000 |
| Maximum balance | 9999 |
| Minimum balance | 0 |
| Balance width | 14 bits |

### 4.6 `balance_to_bcd.v` — Binary to BCD Converter (Shared)
Converts the 14-bit binary balance into four 4-bit BCD digits (thousands, hundreds, tens, ones) using the Double Dabble shift-and-add-3 algorithm. This method is FPGA-safe and avoids division operators which can produce incorrect results during synthesis.

### 4.7 `seven_segment_driver.v` — Display Driver (Shared)
Drives the onboard 4-digit 7-segment display via the MAX7219 SPI chip using a verified state machine (states 0-33 per bit). Supports three display modes:
- Normal: shows balance digits
- Error (`show_err`): displays "Err" using no-decode segment patterns
- Blank (`show_blank`): shuts down the MAX7219 display completely

---

## 5. Build and Run Instructions

### 5.1 Requirements
- Xilinx Vivado 2020.x or later
- Two Anmaya AT-STLN-ARTIX 7-001 boards
- 10 jumper wires + 1 GND wire
- 15V DC power supplies for both boards
- USB-C cables for programming

### 5.2 Project Setup in Vivado

**Board A Project:**
1. Create new Vivado project → RTL Project
2. Add sources: `board_a_top.v`, `debounce.v`, `digit_entry_fsm.v`, `ledger_core.v`, `balance_to_bcd.v`, `seven_segment_driver.v`
3. Set top module: `board_a_top`
4. Add constraints: `board_a.xdc`
5. Run Synthesis → Implementation → Generate Bitstream
6. Program via Hardware Manager → USB-C on J1

**Board B Project:**
1. Create new Vivado project → RTL Project
2. Add sources: `board_b_top.v`, `debounce.v`, `digit_entry_fsm.v`, `ledger_core.v`, `balance_to_bcd.v`, `seven_segment_driver.v`
3. Set top module: `board_b_top`
4. Add constraints: `board_b.xdc`
5. Run Synthesis → Implementation → Generate Bitstream
6. Program via Hardware Manager → USB-C on J1

### 5.3 Hardware Setup

1. Power both boards with 15V DC on J2
2. Connect PMOD jumper wires between boards:

| Wire | Board B Pin | Board A Pin |
|------|------------|------------|
| 1 | IO_0 (T2) | IO_0 (T2) |
| 2 | IO_1 (R3) | IO_1 (R3) |
| 3 | IO_2 (T3) | IO_2 (T3) |
| 4 | IO_3 (T4) | IO_3 (T4) |
| 5 | IO_4 (M1) | IO_4 (M1) |
| 6 | IO_5 (M2) | IO_5 (M2) |
| 7 | IO_6 (P1) | IO_6 (P1) |
| 8 | IO_7 (R1) | IO_7 (R1) |
| 9 | IO_8 (N1) | IO_8 (N1) |
| GND | GND | GND |

3. Set configuration switches: SW17=OFF, SW36=OFF, SW37=ON (Master SPI mode)
4. Power on Board B first, then Board A

### 5.4 Switch Configuration

**Board A:**
| Switch | Pin | Function |
|--------|-----|----------|
| SW0 | C9 | Add mode (deposit) |
| SW1 | B9 | Sub mode (withdrawal) |
| SW2 | G5 | Freeze |
| SW3 | A7 | Shutdown |
| SW4 | C7 | Enable |

**Board B:**
| Switch | Pin | Function |
|--------|-----|----------|
| SW0 | C9 | Reset |
| SW1 | B9 | Add mode (failover only) |
| SW2 | G5 | Sub mode (failover only) |
| SW3 | A7 | Enable (failover only) |

---

## 6. Testing Instructions

### 6.1 Normal Operation Test
1. Power on both boards — Board B LEDs show running light
2. After ~5 seconds Board A LEDs sync with Board B (same running light position)
3. Board A display shows 1000 (initial balance)
4. Board B display is blank (standby mode)
5. On Board A: flip SW1 (add) and SW5 (enable)
6. Press keypad buttons: 0, 5, 0, 0 (enter 0500)
7. Board A display updates to 1500
8. Board B display remains blank but internally stores 1500

### 6.2 Failover Test
1. With Board A showing balance (e.g. 1500)
2. Flip SW4 (shutdown) on Board A
3. Board A display blanks, LEDs turn off
4. Wait 3 seconds — Board B display turns on showing 1500
5. Board B is now primary — use Board B switches to perform transactions
6. Flip SW4 LOW on Board A
7. Board A display shows the latest balance from Board B
8. Board B display blanks — back to standby

### 6.3 Freeze Test
1. Flip SW3 (freeze) on Board A
2. Board A display shows "Err", LEDs freeze at current position
3. Wait 3 seconds — Board B takes over
4. Flip SW3 LOW to release freeze
5. Board A resyncs balance from Board B and resumes

### 6.4 Balance Sync Verification
1. Perform several transactions on Board A
2. Trigger failover (shutdown switch)
3. Perform transactions on Board B
4. Restore Board A (release shutdown)
5. Verify Board A loads Board B's latest balance correctly

---

## 7. Known Constraints

- Maximum balance is 9999 — transactions exceeding this are silently rejected
- Minimum balance is 0 — withdrawals below zero are rejected
- Both add and subtract switches must not be ON simultaneously
- Enable switch must be ON for any transaction to process
- Board B switches (add, sub, enable) have no effect while Board A is alive
- After Board A reset, wait for balance sync from Board B before transacting
