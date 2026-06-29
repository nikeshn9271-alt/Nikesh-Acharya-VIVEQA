# ledger_core.v

**Module:** `ledger_core`
**Role:** Balance register with add, subtract, and forced sync load — shared module used on both Board A and Board B
**Board:** Anmaya AT-STLN-ARTIX 7-001 (XC7A35T-FTG256-1)
**Clock:** 24 MHz

## Description

Maintains a 14-bit balance value (valid range 0–9999). Resets to 1000 on power-on or reset.

**Add:** When enable=1, add_mode=1, sub_mode=0, and value_ready fires, adds entered_value to balance. Uses a 15-bit intermediate sum to detect overflow. Only commits if sum <= 9999.

**Subtract:** When enable=1, sub_mode=1, add_mode=0, and value_ready fires, subtracts entered_value from balance. Only commits if balance >= entered_value (prevents underflow below zero).

**Sync load (sync_in):** Overrides normal operation. When sync_in = 1, immediately loads sync_val into balance regardless of enable or value_ready. Used by Board A to load balance received from Board B after freeze/shutdown release. Used by Board B to load Board A's balance on reconnect.

**tx_done:** Pulses HIGH for exactly 1 clock cycle when a transaction is successfully committed. Used by digit_entry_fsm to reset after a completed transaction.

## Ports
| Port | Direction | Width | Description |
|------|-----------|-------|-------------|
| clk | Input | 1 | 24 MHz system clock |
| rst | Input | 1 | Asynchronous reset — sets balance to 1000 |
| enable | Input | 1 | Must be HIGH for transactions to process |
| add_mode | Input | 1 | Addition mode |
| sub_mode | Input | 1 | Subtraction mode |
| value_ready | Input | 1 | 1-cycle pulse from digit_entry_fsm on 4th digit |
| entered_value | Input | 14 | Amount entered by user |
| sync_in | Input | 1 | Force load sync_val into balance immediately |
| sync_val | Input | 14 | Balance value to load when sync_in fires |
| balance | Output (reg) | 14 | Current balance |
| tx_done | Output (reg) | 1 | 1-cycle pulse when transaction committed |

```verilog
module ledger_core (
    input             clk,
    input             rst,
    input             enable,
    input             add_mode,
    input             sub_mode,
    input             value_ready,
    input      [13:0] entered_value,
    input             sync_in,
    input      [13:0] sync_val,
    output reg [13:0] balance,
    output reg        tx_done
);
    wire [14:0] sum = {1'b0, balance} + {1'b0, entered_value};
    always @(posedge clk or posedge rst) begin
        if (rst) begin
            balance <= 14'd1000;
            tx_done <= 1'b0;
        end else if (sync_in) begin
            balance <= sync_val;
            tx_done <= 1'b0;
        end else begin
            tx_done <= 1'b0;
            if (enable && value_ready) begin
                if (add_mode && !sub_mode && (sum <= 15'd9999)) begin
                    balance <= sum[13:0];
                    tx_done <= 1'b1;
                end else if (sub_mode && !add_mode && (balance >= entered_value)) begin
                    balance <= balance - entered_value;
                    tx_done <= 1'b1;
                end
            end
        end
    end
endmodule
```
