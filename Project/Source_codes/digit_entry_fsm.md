# digit_entry_fsm.v

**Module:** `digit_entry_fsm`
**Role:** 4-digit entry state machine — shared module used on both Board A and Board B
**Board:** Anmaya AT-STLN-ARTIX 7-001 (XC7A35T-FTG256-1)
**Clock:** 24 MHz

## Description

Manages digit-by-digit amount entry from the keypad. User must press exactly 4 buttons in sequence to complete a transaction. The FSM fires value_ready on the 4th press and the entered amount is passed to ledger_core.

**Button mapping:** btn[0]=0, btn[1]=1, btn[2]=2, ..., btn[9]=9. To enter 500 press 0, 5, 0, 0. To enter 1234 press 1, 2, 3, 4.

**valid_mode:** XOR of add_mode and sub_mode. Both HIGH or both LOW = invalid, FSM stays in reset. Exactly one of add_mode or sub_mode must be HIGH for entry to work.

**Auto-reset conditions:** FSM resets immediately when valid_mode goes LOW (switch released), when tx_done fires (transaction committed), or when mode_activated fires (mode switch just turned on).

**entry_active:** Goes HIGH on first digit press. Stays HIGH until reset. Can be used externally to switch display from balance to entry digits.

**disp_thousands/hundreds/tens/ones:** Outputs current entered digits for display. Updates as each digit is pressed.

## Ports
| Port | Direction | Width | Description |
|------|-----------|-------|-------------|
| clk | Input | 1 | 24 MHz system clock |
| rst | Input | 1 | Asynchronous reset |
| add_mode | Input | 1 | Addition mode active (gated by enable, freeze, shutdown) |
| sub_mode | Input | 1 | Subtraction mode active (gated by enable, freeze, shutdown) |
| db_btn | Input | 10 | Debounced button pulses from btn[0]–btn[9] |
| tx_done | Input | 1 | Transaction committed — resets FSM |
| entered_value | Output (reg) | 14 | Completed 4-digit amount (valid when value_ready=1) |
| value_ready | Output (reg) | 1 | 1-cycle pulse after 4th digit pressed |
| disp_thousands | Output (reg) | 4 | First digit entered |
| disp_hundreds | Output (reg) | 4 | Second digit entered |
| disp_tens | Output (reg) | 4 | Third digit entered |
| disp_ones | Output (reg) | 4 | Fourth digit entered |
| entry_active | Output (reg) | 1 | HIGH after first digit pressed |

```verilog
module digit_entry_fsm (
    input             clk,
    input             rst,
    input             add_mode,
    input             sub_mode,
    input      [9:0]  db_btn,
    input             tx_done,
    output reg [13:0] entered_value,
    output reg        value_ready,
    output reg [3:0]  disp_thousands,
    output reg [3:0]  disp_hundreds,
    output reg [3:0]  disp_tens,
    output reg [3:0]  disp_ones,
    output reg        entry_active
);
    reg [1:0] digit_count;
    reg [3:0] d0, d1, d2;

    wire valid_mode = add_mode ^ sub_mode;
    wire any_press  = |db_btn;
    reg valid_mode_prev;
    wire mode_activated = valid_mode & ~valid_mode_prev;

    wire [3:0] pressed_digit =
        db_btn[1] ? 4'd1 : db_btn[2] ? 4'd2 : db_btn[3] ? 4'd3 :
        db_btn[4] ? 4'd4 : db_btn[5] ? 4'd5 : db_btn[6] ? 4'd6 :
        db_btn[7] ? 4'd7 : db_btn[8] ? 4'd8 : db_btn[9] ? 4'd9 : 4'd0;

    task do_reset;
        begin
            digit_count    <= 2'd0;
            d0 <= 0; d1 <= 0; d2 <= 0;
            entered_value  <= 14'd0;
            value_ready    <= 1'b0;
            entry_active   <= 1'b0;
            disp_thousands <= 4'd0;
            disp_hundreds  <= 4'd0;
            disp_tens      <= 4'd0;
            disp_ones      <= 4'd0;
        end
    endtask

    always @(posedge clk or posedge rst) begin
        if (rst) begin
            valid_mode_prev <= 1'b0;
            do_reset;
        end else begin
            valid_mode_prev <= valid_mode;
            value_ready     <= 1'b0;

            if (mode_activated || tx_done || !valid_mode) begin
                do_reset;
            end else if (valid_mode && any_press) begin
                entry_active <= 1'b1;
                case (digit_count)
                    2'd0: begin
                        d0             <= pressed_digit;
                        disp_thousands <= pressed_digit;
                        digit_count    <= 2'd1;
                    end
                    2'd1: begin
                        d1             <= pressed_digit;
                        disp_thousands <= d0;
                        disp_hundreds  <= pressed_digit;
                        digit_count    <= 2'd2;
                    end
                    2'd2: begin
                        d2             <= pressed_digit;
                        disp_thousands <= d0;
                        disp_hundreds  <= d1;
                        disp_tens      <= pressed_digit;
                        digit_count    <= 2'd3;
                    end
                    2'd3: begin
                        disp_thousands <= d0;
                        disp_hundreds  <= d1;
                        disp_tens      <= d2;
                        disp_ones      <= pressed_digit;
                        entered_value  <= (d0 * 14'd1000) + (d1 * 14'd100) + (d2 * 14'd10) + pressed_digit;
                        value_ready    <= 1'b1;
                        digit_count    <= 2'd0;
                    end
                endcase
            end
        end
    end
endmodule
```
