# board_b_top.v

**Module:** `board_b_top`
**Role:** Board B — Backup FPGA board and Heartbeat Master
**Board:** Anmaya AT-STLN-ARTIX 7-001 (XC7A35T-FTG256-1)
**Clock:** 24 MHz (D13)

## Switch Map
| Switch | Pin | Function |
|--------|-----|----------|
| sw[0] | C9 | Reset (active LOW — sw[0] LOW = rst HIGH) |
| sw[1] | B9 | Add mode (active LOW, only works in failover) |
| sw[2] | G5 | Sub mode (active LOW, only works in failover) |
| sw[3] | A7 | Enable transactions (active LOW, must be LOW to allow transactions in failover) |

## PMOD Pin Map
| Signal | Pin | Direction | Function |
|--------|-----|-----------|----------|
| pmod_hb_pos[0] | T2 | Output | Heartbeat position bit 0 to Board A |
| pmod_hb_pos[1] | R3 | Output | Heartbeat position bit 1 to Board A |
| pmod_hb_pos[2] | T3 | Output | Heartbeat position bit 2 to Board A |
| pmod_hb_tick | T4 | Output | Heartbeat tick pulse to Board A (1 cycle every 500ms) |
| pmod_sync_data | R1 | Output | Balance serial data B→A |
| pmod_sync_clk | N1 | Output | Balance serial clock B→A |
| pmod_ack_valid | M1 | Input | Board A alive signal (continuous HIGH when A is running) |
| pmod_bal_data | M2 | Input | Balance serial data A→B |
| pmod_bal_clk | P1 | Input | Balance serial clock A→B |

## Functional Description

**Heartbeat master:** Generates a 3-bit counter advancing every 500ms (HALF_SEC = 11,999,999 cycles at 24 MHz). Drives 8 LEDs as one-hot running light. Sends hb_pos[2:0] and a 1-cycle hb_tick pulse to Board A continuously via PMOD.

**Grace period:** 5-second startup timer (GRACE_5S = 119,999,999 cycles). Timeout counting and standalone boot bypass do not activate until grace_done goes HIGH. Prevents Board B from falsely activating before Board A has had time to start up.

**ACK monitor:** 4-bit shift filter on pmod_ack_valid. Requires 4 consecutive HIGH samples (4'b1111) before board_a_alive goes HIGH. Board A drives pmod_ack_valid continuously HIGH while alive. If ACK is absent for 3 seconds (TIMEOUT_3S = 71,999,999) after Board A was previously seen, board_a_alive goes LOW.

**board_a_seen flag:** Latches HIGH permanently on first confirmed ACK. Timeout counting only begins after board_a_seen = 1, so Board B never activates unless Board A was alive at least once first.

**Standalone boot bypass:** If board_a_seen is still 0 after grace_done (Board A never connected), board_b_active goes HIGH anyway so Board B can operate independently.

**Failover logic:** board_b_active goes HIGH when Board A is confirmed dead after being seen. Goes LOW immediately when Board A comes back (board_a_alive = 1). Transactions gated by b_enabled = board_b_active AND sw_enable.

**A→B receiver:** 2-stage synchroniser on pmod_bal_clk and pmod_bal_data. Detects sync_start pulse then receives 14 data bits MSB first into received_balance. Updates on every complete frame (~30ms) so Board B always has Board A's latest balance.

**board_a_returned:** 1-cycle pulse on rising edge of board_a_alive. Triggers B→A sync send and ledger reload.

**B→A sync sender:** Sends balance to Board A when board_a_returned fires (immediate reconnect sync) or periodic_send fires (every 1 second). When board_b_active = 1 (failover), sends balance_B (Board B's own ledger). When board_b_active = 0 (standby), sends received_balance (mirrors Board A's balance back to it). Same framed protocol as A→B sender.

**Ledger core:** sync_in loads received_balance on board_a_returned (reconnect) or whenever a new frame completes while not in failover mode and Board A is in standby. This keeps Board B's ledger always tracking Board A's balance.

**Display:** Blank when board_b_active = 0 (Board A alive). Shows balance when board_b_active = 1 (failover active). Never shows Err.

## Serial Protocol (B→A)
- Clock: SYNC_HALF = 23999 → tick every 24000 cycles = 1ms at 24 MHz
- Each bit = 2 ticks = 2ms
- Frame = 1 sync_start + 14 data bits = 30ms total
- Periodic send interval: PERIODIC_1S = 23,999,999 cycles = 1 second

```verilog
// =============================================================================
// board_b_top.v  - Board B (BACKUP)
// sw[0]=rst  sw[1]=add  sw[2]=sub  sw[3]=enable
// =============================================================================
module board_b_top (
    input        clk,
    input  [3:0] sw,
    input  [15:0] btn,
    output [7:0] led,
    output       seg_din,
    output       seg_clk,
    output       seg_cs,
    output [2:0] pmod_hb_pos,
    output       pmod_hb_tick,
    input        pmod_ack_valid,
    input        pmod_bal_data,
    input        pmod_bal_clk,
    output       pmod_sync_data,
    output       pmod_sync_clk
);
    wire rst       = ~sw[0];
    wire sw_add    = ~sw[1];
    wire sw_sub    = ~sw[2];
    wire sw_enable = ~sw[3];

    reg [23:0] hb_counter;
    reg [2:0]  hb_pos;
    reg        hb_tick;

    parameter HALF_SEC = 24'd11_999_999;

    always @(posedge clk or posedge rst) begin
        if (rst) begin
            hb_counter <= 24'd0;
            hb_pos     <= 3'd0;
            hb_tick    <= 1'b0;
        end else begin
            if (hb_counter >= HALF_SEC) begin
                hb_counter <= 24'd0;
                hb_tick    <= 1'b1;
                hb_pos     <= hb_pos + 1'b1;
            end else begin
                hb_counter <= hb_counter + 1'b1;
                hb_tick    <= 1'b0;
            end
        end
    end

    assign pmod_hb_pos  = hb_pos;
    assign pmod_hb_tick = hb_tick;
    assign led = (8'b00000001 << hb_pos);

    parameter TIMEOUT_3S = 27'd71_999_999;
    parameter GRACE_5S   = 28'd119_999_999;

    reg [27:0] grace_cnt;
    reg        grace_done;

    always @(posedge clk or posedge rst) begin
        if (rst) begin
            grace_cnt  <= 28'd0;
            grace_done <= 1'b0;
        end else if (!grace_done) begin
            if (grace_cnt >= GRACE_5S)
                grace_done <= 1'b1;
            else
                grace_cnt <= grace_cnt + 1'b1;
        end
    end

    reg [26:0] timeout_cnt;
    reg [3:0]  ack_filter;
    reg        board_a_alive;
    reg        board_a_seen;

    always @(posedge clk or posedge rst) begin
        if (rst) begin
            timeout_cnt   <= 27'd0;
            ack_filter    <= 4'b0000;
            board_a_alive <= 1'b0;
            board_a_seen  <= 1'b0;
        end else begin
            ack_filter <= {ack_filter[2:0], pmod_ack_valid};
            if (ack_filter == 4'b1111) begin
                timeout_cnt   <= 27'd0;
                board_a_alive <= 1'b1;
                board_a_seen  <= 1'b1;
            end else begin
                if (board_a_seen) begin
                    if (timeout_cnt < TIMEOUT_3S)
                        timeout_cnt <= timeout_cnt + 1'b1;
                    else
                        board_a_alive <= 1'b0;
                end
            end
        end
    end

    reg board_b_active;
    always @(posedge clk or posedge rst) begin
        if (rst) begin
            board_b_active <= 1'b0;
        end else begin
            if (board_a_alive) begin
                board_b_active <= 1'b0;
            end else if (board_a_seen && !board_a_alive) begin
                board_b_active <= 1'b1;
            end else if (!board_a_seen && grace_done) begin
                board_b_active <= 1'b1; // Standalone boot bypass
            end
        end
    end

    wire b_enabled = board_b_active & sw_enable;

    reg bal_clk_sync0, bal_clk_sync1, bal_clk_prev;
    reg bal_data_sync0, bal_data_sync;

    always @(posedge clk or posedge rst) begin
        if (rst) begin
            bal_clk_sync0  <= 0;
            bal_clk_sync1  <= 0;
            bal_data_sync0 <= 0; bal_data_sync  <= 0;
        end else begin
            bal_clk_sync0  <= pmod_bal_clk;
            bal_clk_sync1  <= bal_clk_sync0;
            bal_data_sync0 <= pmod_bal_data;
            bal_data_sync  <= bal_data_sync0;
        end
    end

    always @(posedge clk or posedge rst) begin
        if (rst) bal_clk_prev <= 0;
        else     bal_clk_prev <= bal_clk_sync1;
    end

    wire bal_clk_rise = bal_clk_sync1 & ~bal_clk_prev;

    reg [3:0]  recv_bit_cnt;
    reg [13:0] recv_shift;
    reg [13:0] received_balance;
    reg        frame_active;

    always @(posedge clk or posedge rst) begin
        if (rst) begin
            recv_bit_cnt     <= 4'd13;
            recv_shift       <= 14'd0;
            received_balance <= 14'd1000;
            frame_active     <= 1'b0;
        end else begin
            if (bal_clk_rise) begin
                if (bal_data_sync && !frame_active) begin
                    recv_bit_cnt <= 4'd13;
                    recv_shift   <= 14'd0;
                    frame_active <= 1'b1;
                end else if (frame_active) begin
                    recv_shift <= {recv_shift[12:0], bal_data_sync};
                    if (recv_bit_cnt == 4'd0) begin
                        received_balance <= {recv_shift[12:0], bal_data_sync};
                        recv_bit_cnt     <= 4'd13;
                        frame_active     <= 1'b0;
                    end else begin
                        recv_bit_cnt <= recv_bit_cnt - 1'b1;
                    end
                end
            end
        end
    end

    reg board_a_alive_prev;
    wire board_a_returned = board_a_alive & ~board_a_alive_prev;

    always @(posedge clk or posedge rst) begin
        if (rst) board_a_alive_prev <= 1'b0;
        else     board_a_alive_prev <= board_a_alive;
    end

    reg [26:0] sync_tick_cnt;
    parameter  SYNC_HALF = 27'd23999;
    wire sync_tick = (sync_tick_cnt == SYNC_HALF);

    always @(posedge clk or posedge rst) begin
        if (rst) sync_tick_cnt <= 27'd0;
        else if (sync_tick) sync_tick_cnt <= 27'd0;
        else sync_tick_cnt <= sync_tick_cnt + 1'b1;
    end

    reg [26:0] periodic_cnt;
    parameter  PERIODIC_1S = 27'd23_999_999;
    reg        periodic_send;

    always @(posedge clk or posedge rst) begin
        if (rst) begin
            periodic_cnt  <= 27'd0;
            periodic_send <= 1'b0;
        end else begin
            periodic_send <= 1'b0;
            if (periodic_cnt >= PERIODIC_1S) begin
                periodic_cnt  <= 27'd0;
                periodic_send <= 1'b1;
            end else begin
                periodic_cnt <= periodic_cnt + 1'b1;
            end
        end
    end

    localparam SYNC_IDLE      = 2'd0;
    localparam SYNC_SEND_SYNC = 2'd1;
    localparam SYNC_SEND_BITS = 2'd2;

    reg [1:0]  sync_state;
    reg [3:0]  sync_bit_cnt;
    reg [13:0] sync_shift;
    reg        sync_phase;
    reg        sync_data_reg;
    reg        sync_clk_reg;
    
    wire [13:0] balance_B;

    always @(posedge clk or posedge rst) begin
        if (rst) begin
            sync_state    <= SYNC_IDLE;
            sync_bit_cnt  <= 4'd13;
            sync_shift    <= 14'd0;
            sync_phase    <= 1'b0;
            sync_data_reg <= 1'b0;
            sync_clk_reg  <= 1'b0;
        end else begin
            case (sync_state)
                SYNC_IDLE: begin
                    sync_data_reg <= 1'b0;
                    sync_clk_reg  <= 1'b0;
                    if (board_a_returned || periodic_send) begin
                        sync_shift   <= board_b_active ? balance_B : received_balance;
                        sync_bit_cnt <= 4'd13;
                        sync_phase   <= 1'b0;
                        sync_state   <= SYNC_SEND_SYNC;
                    end
                end

                SYNC_SEND_SYNC: begin
                    if (sync_tick) begin
                        if (sync_phase == 1'b0) begin
                            sync_data_reg <= 1'b1;
                            sync_clk_reg  <= 1'b1;
                            sync_phase    <= 1'b1;
                        end else begin
                            sync_clk_reg  <= 1'b0;
                            sync_phase    <= 1'b0;
                            sync_state    <= SYNC_SEND_BITS;
                        end
                    end
                end

                SYNC_SEND_BITS: begin
                    if (sync_tick) begin
                        if (sync_phase == 1'b0) begin
                            sync_data_reg <= sync_shift[13];
                            sync_clk_reg  <= 1'b0;
                            sync_phase    <= 1'b1;
                        end else begin
                            sync_clk_reg  <= 1'b1;
                            sync_shift    <= {sync_shift[12:0], 1'b0};
                            if (sync_bit_cnt == 4'd0)
                                sync_state <= SYNC_IDLE;
                            else
                                sync_bit_cnt <= sync_bit_cnt - 1'b1;
                            sync_phase <= 1'b0;
                        end
                    end
                end
                default: sync_state <= SYNC_IDLE;
            endcase
        end
    end

    assign pmod_sync_data = sync_data_reg;
    assign pmod_sync_clk  = sync_clk_reg;

    wire [9:0] db_btn;

    generate
        for (genvar i = 0; i < 10; i = i + 1) begin: deb
            debounce U_DB (
                .clk(clk), .rst(rst),
                .btn(btn[i] & b_enabled),
                .pulse(db_btn[i])
            );
        end
    endgenerate

    wire [13:0] entered_value;
    wire        value_ready;
    wire        tx_done;

    digit_entry_fsm U_ENTRY (
        .clk           (clk),
        .rst           (rst),
        .add_mode      (sw_add & b_enabled),
        .sub_mode      (sw_sub & b_enabled),
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

    ledger_core U_LEDGER (
        .clk          (clk),
        .rst          (rst),
        .enable       (b_enabled),
        .add_mode     (sw_add),
        .sub_mode     (sw_sub),
        .value_ready  (value_ready),
        .entered_value(entered_value),
        .sync_in      (board_a_returned | (!board_b_active & bal_clk_rise & ~frame_active)), 
        .sync_val     (received_balance),
        .balance      (balance_B),
        .tx_done      (tx_done)
    );

    wire [3:0] bal_th, bal_hu, bal_te, bal_on;

    balance_to_bcd U_BCD (
        .balance  (balance_B),
        .thousands(bal_th), .hundreds(bal_hu),
        .tens     (bal_te), .ones    (bal_on)
    );

    seven_segment_driver U_SEG (
        .clk       (clk), .rst(rst),
        .thousands (bal_th),
        .hundreds  (bal_hu),
        .tens      (bal_te),
        .ones      (bal_on),
        .show_err  (1'b0),
        .show_blank(~board_b_active),
        .seg_din   (seg_din),
        .seg_clk   (seg_clk),
        .seg_cs    (seg_cs)
    );
endmodule
```
