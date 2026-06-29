# board_a_top.v

**Module:** `board_a_top`
**Role:** Board A — Primary FPGA board
**Board:** Anmaya AT-STLN-ARTIX 7-001 (XC7A35T-FTG256-1)
**Clock:** 24 MHz (D13)

## Switch Map
| Switch | Pin | Function |
|--------|-----|----------|
| sw[0] | C9 | Add mode (active LOW) |
| sw[1] | B9 | Sub mode (active LOW) |
| sw[2] | G5 | Freeze — stops ACK, shows Err on display (active LOW) |
| sw[3] | A7 | Shutdown — simulates power off (active LOW) |
| sw[4] | C7 | Enable — must be LOW to allow transactions (active LOW) |

> No reset switch. Reset is hardwired LOW (rst = 1'b0). Power cycle to reset.

## PMOD Pin Map
| Signal | Pin | Direction | Function |
|--------|-----|-----------|----------|
| pmod_ack_valid | M1 | Output | Continuous HIGH while Board A is alive and not frozen/shutdown |
| pmod_bal_data | M2 | Output | Balance serial data A→B |
| pmod_bal_clk | P1 | Output | Balance serial clock A→B |
| pmod_hb_pos[0] | T2 | Input | Heartbeat position bit 0 from Board B |
| pmod_hb_pos[1] | R3 | Input | Heartbeat position bit 1 from Board B |
| pmod_hb_pos[2] | T3 | Input | Heartbeat position bit 2 from Board B |
| pmod_hb_tick | T4 | Input | Heartbeat tick pulse from Board B |
| pmod_sync_data | R1 | Input | Balance serial data B→A |
| pmod_sync_clk | N1 | Input | Balance serial clock B→A |

## Functional Description

**Heartbeat slave:** Receives 3-bit heartbeat position from Board B via PMOD. Mirrors it as a running light on 8 LEDs. Freezes LED position when freeze_active is HIGH. Clears LEDs when shutdown is active.

**ACK signal:** Drives pmod_ack_valid continuously HIGH while Board A is alive (not frozen, not shutdown). Board B uses this to confirm Board A is running. Goes LOW during freeze or shutdown so Board B takes over after 3 seconds.

**B→A sync receiver:** 2-stage synchroniser on pmod_sync_clk and pmod_sync_data. Detects sync_start pulse (rising edge with data=1) then receives 14 data bits MSB first. Fires srx_valid for 1 cycle when a complete frame is received.

**wait_sync flag:** Goes HIGH when freeze or shutdown releases (falling edge). Blocks transactions and loads balance from Board B when srx_valid fires. Cleared immediately after sync received.

**Digit entry:** 10 buttons (btn[0]–btn[9]) debounced and fed to digit_entry_fsm. Exactly 4 digits must be entered. value_ready fires on 4th press. Gated by freeze, shutdown, wait_sync, and enable.

**Ledger core:** Maintains 14-bit balance (0–9999). Adds or subtracts entered_value on value_ready. sync_in loads sync_received_balance when wait_sync is HIGH and srx_valid fires.

**A→B serial sender:** Continuously sends current balance to Board B every ~30ms using framed serial protocol (sync_start pulse + 14 data bits MSB first, 2ms per bit at 24 MHz). On freeze or shutdown rising edge, immediately captures and sends the current balance one final time (force_send) before going silent. Never aborts mid-frame.

**Display:** balance_to_bcd converts balance to 4 BCD digits. seven_segment_driver sends to MAX7219 via SPI. Shows Err when frozen. Blank when shutdown.

## Serial Protocol (A→B)
- Clock: BAL_HALF = 23999 → tick every 24000 cycles = 1ms at 24 MHz
- Each bit = 2 ticks = 2ms
- Frame = 1 sync_start + 14 data bits = 30ms total
- sync_start: data=1 and clock rise simultaneously
- Data bits: data set in phase=0, clock rises in phase=1

```verilog
// =============================================================================
// board_a_top.v  -  Board A (PRIMARY)
// sw[0]=add  sw[1]=sub  sw[2]=freeze
// sw[3]=shutdown  sw[4]=enable
// rst = 1'b0 (no reset switch - power cycle to reset)
// =============================================================================
module board_a_top (
    input        clk,
    input  [4:0] sw,
    input  [15:0] btn,
    output [7:0] led,
    output       seg_din,
    output       seg_clk,
    output       seg_cs,
    output       pmod_ack_valid,
    output       pmod_bal_data,
    output       pmod_bal_clk,
    input  [2:0] pmod_hb_pos,
    input        pmod_hb_tick,
    input        pmod_sync_data,
    input        pmod_sync_clk
);
   wire rst         = 1'b0;
   wire sw_add      = ~sw[0];
   wire sw_sub      = ~sw[1];
   wire sw_freeze   = ~sw[2];
   wire sw_shutdown = ~sw[3];
   wire sw_enable   = ~sw[4];

    wire freeze_active = sw_freeze;

    reg [2:0] hb_pos_A;
    always @(posedge clk or posedge rst) begin
        if (rst)
            hb_pos_A <= 3'd0;
        else if (sw_shutdown)
            hb_pos_A <= 3'd0;
        else if (freeze_active)
            hb_pos_A <= hb_pos_A;
        else begin
            if (pmod_hb_tick)
                hb_pos_A <= pmod_hb_pos;
        end
    end

    assign led = sw_shutdown ? 8'b00000000 : (8'b00000001 << hb_pos_A);
    assign pmod_ack_valid = (~sw_shutdown & ~freeze_active);

    reg sync_clk_s0, sync_clk_s1, sync_clk_prev;
    reg sync_data_s0, sync_data_s;
    always @(posedge clk or posedge rst) begin
        if (rst) begin
            sync_clk_s0  <= 0; sync_clk_s1  <= 0;
            sync_data_s0 <= 0; sync_data_s  <= 0;
        end else begin
            sync_clk_s0  <= pmod_sync_clk;
            sync_clk_s1  <= sync_clk_s0;
            sync_data_s0 <= pmod_sync_data;
            sync_data_s  <= sync_data_s0;
        end
    end

    always @(posedge clk or posedge rst) begin
        if (rst) sync_clk_prev <= 0;
        else     sync_clk_prev <= sync_clk_s1;
    end

    wire sync_clk_rise = sync_clk_s1 & ~sync_clk_prev;

    reg [3:0]  srx_bit_cnt;
    reg [13:0] srx_shift;
    reg [13:0] sync_received_balance;
    reg        srx_frame_active;
    reg        srx_valid;

    always @(posedge clk or posedge rst) begin
        if (rst) begin
            srx_bit_cnt           <= 4'd13;
            srx_shift             <= 14'd0;
            sync_received_balance <= 14'd1000;
            srx_frame_active      <= 1'b0;
            srx_valid             <= 1'b0;
        end else begin
            srx_valid <= 1'b0;
            if (sync_clk_rise) begin
                if (sync_data_s && !srx_frame_active) begin
                    srx_bit_cnt      <= 4'd13;
                    srx_shift        <= 14'd0;
                    srx_frame_active <= 1'b1;
                end else if (srx_frame_active) begin
                    srx_shift <= {srx_shift[12:0], sync_data_s};
                    if (srx_bit_cnt == 4'd0) begin
                        sync_received_balance <= {srx_shift[12:0], sync_data_s};
                        srx_frame_active      <= 1'b0;
                        srx_valid             <= 1'b1;
                    end else begin
                        srx_bit_cnt <= srx_bit_cnt - 1'b1;
                    end
                end
            end
        end
    end

    reg sw_freeze_prev;
    reg sw_shutdown_prev;

    wire freeze_rising   = sw_freeze   & ~sw_freeze_prev;
    wire shutdown_rising = sw_shutdown & ~sw_shutdown_prev;
    wire freeze_falling  = sw_freeze_prev   & ~sw_freeze;
    wire shutdown_falling = sw_shutdown_prev & ~sw_shutdown;

    wire force_send = freeze_rising | shutdown_rising;

    always @(posedge clk or posedge rst) begin
        if (rst) begin
            sw_freeze_prev   <= 1'b0;
            sw_shutdown_prev <= 1'b0;
        end else begin
            sw_freeze_prev   <= sw_freeze;
            sw_shutdown_prev <= sw_shutdown;
        end
    end

    reg wait_sync;
    always @(posedge clk or posedge rst) begin
        if (rst) begin
            wait_sync <= 1'b0;
        end else begin
            if (srx_valid && wait_sync)
                wait_sync <= 1'b0;
            if (freeze_falling || shutdown_falling)
                wait_sync <= 1'b1;
        end
    end

    wire [9:0] db_btn;
    generate
        for (genvar i = 0; i < 10; i = i + 1) begin: deb
            debounce U_DB (.clk(clk), .rst(rst), .btn(btn[i]), .pulse(db_btn[i]));
        end
    endgenerate

    wire [13:0] entered_value;
    wire        value_ready;
    wire        tx_done;

    digit_entry_fsm U_ENTRY (
        .clk           (clk),
        .rst           (rst),
        .add_mode      (sw_add  & ~freeze_active & ~sw_shutdown & ~wait_sync & sw_enable),
        .sub_mode      (sw_sub  & ~freeze_active & ~sw_shutdown & ~wait_sync & sw_enable),
        .db_btn        (db_btn),
        .tx_done       (tx_done),
        .entered_value (entered_value),
        .value_ready   (value_ready),
        .disp_thousands(),
        .disp_hundreds (),
        .disp_tens     (),
        .disp_ones     (),
        .entry_active  ()
    );

    wire [13:0] balance;

    ledger_core U_LEDGER (
        .clk          (clk),
        .rst          (rst),
        .enable       (~freeze_active & ~sw_shutdown & ~wait_sync & sw_enable),
        .add_mode     (sw_add),
        .sub_mode     (sw_sub),
        .value_ready  (value_ready),
        .entered_value(entered_value),
        .sync_in      (srx_valid & wait_sync),
        .sync_val     (sync_received_balance),
        .balance      (balance),
        .tx_done      (tx_done)
    );

    reg [25:0] bal_tick_cnt;
    parameter  BAL_HALF = 26'd23999;
    wire bal_tick = (bal_tick_cnt == BAL_HALF);

    always @(posedge clk or posedge rst) begin
        if (rst) bal_tick_cnt <= 26'd0;
        else if (bal_tick) bal_tick_cnt <= 26'd0;
        else bal_tick_cnt <= bal_tick_cnt + 1'b1;
    end

    localparam BAL_IDLE      = 2'd0;
    localparam BAL_SEND_SYNC = 2'd1;
    localparam BAL_SEND_BITS = 2'd2;

    reg [1:0]  bal_state;
    reg [3:0]  bal_bit_cnt;
    reg [13:0] bal_shift;
    reg        bal_phase;
    reg        bal_data_reg;
    reg        bal_clk_reg;
    reg        forced_send_done;

    always @(posedge clk or posedge rst) begin
        if (rst) begin
            bal_state       <= BAL_IDLE;
            bal_bit_cnt     <= 4'd13;
            bal_shift       <= 14'd0;
            bal_phase       <= 1'b0;
            bal_data_reg    <= 1'b0;
            bal_clk_reg     <= 1'b0;
            forced_send_done <= 1'b0;
        end else begin

            if (!sw_freeze && !sw_shutdown)
                forced_send_done <= 1'b0;

            case (bal_state)
                BAL_IDLE: begin
                    bal_data_reg <= 1'b0;
                    bal_clk_reg  <= 1'b0;

                    if (force_send && !forced_send_done) begin
                        bal_shift        <= balance;
                        bal_bit_cnt      <= 4'd13;
                        bal_phase        <= 1'b0;
                        forced_send_done <= 1'b1;
                        bal_state        <= BAL_SEND_SYNC;

                    end else if (bal_tick && !sw_shutdown && !freeze_active) begin
                        bal_shift    <= balance;
                        bal_bit_cnt  <= 4'd13;
                        bal_phase    <= 1'b0;
                        bal_state    <= BAL_SEND_SYNC;
                    end
                end

                BAL_SEND_SYNC: begin
                    if (bal_tick) begin
                        if (bal_phase == 1'b0) begin
                            bal_data_reg <= 1'b1;
                            bal_clk_reg  <= 1'b1;
                            bal_phase    <= 1'b1;
                        end else begin
                            bal_clk_reg  <= 1'b0;
                            bal_phase    <= 1'b0;
                            bal_state    <= BAL_SEND_BITS;
                        end
                    end
                end

                BAL_SEND_BITS: begin
                    if (bal_tick) begin
                        if (bal_phase == 1'b0) begin
                            bal_data_reg <= bal_shift[13];
                            bal_clk_reg  <= 1'b0;
                            bal_phase    <= 1'b1;
                        end else begin
                            bal_clk_reg  <= 1'b1;
                            bal_shift    <= {bal_shift[12:0], 1'b0};
                            if (bal_bit_cnt == 4'd0)
                                bal_state <= BAL_IDLE;
                            else
                                bal_bit_cnt <= bal_bit_cnt - 1'b1;
                            bal_phase <= 1'b0;
                        end
                    end
                end

                default: bal_state <= BAL_IDLE;
            endcase
        end
    end

    assign pmod_bal_data = bal_data_reg;
    assign pmod_bal_clk  = bal_clk_reg;

    wire [3:0] bal_th, bal_hu, bal_te, bal_on;

    balance_to_bcd U_BCD (
        .balance  (balance),
        .thousands(bal_th), .hundreds(bal_hu),
        .tens     (bal_te), .ones    (bal_on)
    );

    seven_segment_driver U_SEG (
        .clk       (clk), .rst(rst),
        .thousands (bal_th),
        .hundreds  (bal_hu),
        .tens      (bal_te),
        .ones      (bal_on),
        .show_err  (freeze_active),
        .show_blank(sw_shutdown),
        .seg_din   (seg_din),
        .seg_clk   (seg_clk),
        .seg_cs    (seg_cs)
    );

endmodule
```
