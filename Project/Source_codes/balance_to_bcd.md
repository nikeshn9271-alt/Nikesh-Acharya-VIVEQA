# balance_to_bcd.v

**Module:** `balance_to_bcd`
**Role:** Binary to 4-digit BCD converter — shared module used on both Board A and Board B
**Board:** Anmaya AT-STLN-ARTIX 7-001 (XC7A35T-FTG256-1)

## Description

Converts a 14-bit binary balance value (0–9999) into four separate 4-bit BCD digits (thousands, hundreds, tens, ones) using the Double Dabble algorithm.

**Double Dabble** is the standard FPGA-safe BCD conversion method. It avoids integer division which produces incorrect results in synthesis. The algorithm shifts the binary value left one bit at a time through 14 iterations. Before each shift, any BCD digit that is >= 5 has 3 added to it. After 14 shifts the upper 16 bits contain the four BCD digits.

**Purely combinational** — no clock, no registers, no latency. Output updates immediately whenever balance changes.

**Input range:** 0 to 9999 (14-bit). Values above 9999 should never reach this module as ledger_core clamps the balance.

## Ports
| Port | Direction | Width | Description |
|------|-----------|-------|-------------|
| balance | Input | 14 | Binary balance value (0–9999) |
| thousands | Output (reg) | 4 | Thousands digit (0–9) |
| hundreds | Output (reg) | 4 | Hundreds digit (0–9) |
| tens | Output (reg) | 4 | Tens digit (0–9) |
| ones | Output (reg) | 4 | Ones digit (0–9) |

```verilog
module balance_to_bcd (
    input      [13:0] balance,
    output reg [3:0]  thousands,
    output reg [3:0]  hundreds,
    output reg [3:0]  tens,
    output reg [3:0]  ones
);
    integer k;
    reg [29:0] shift_reg;
    always @(*) begin
        shift_reg = {16'b0, balance};
        for (k = 0; k < 14; k = k + 1) begin
            if (shift_reg[17:14] >= 5) shift_reg[17:14] = shift_reg[17:14] + 3;
            if (shift_reg[21:18] >= 5) shift_reg[21:18] = shift_reg[21:18] + 3;
            if (shift_reg[25:22] >= 5) shift_reg[25:22] = shift_reg[25:22] + 3;
            if (shift_reg[29:26] >= 5) shift_reg[29:26] = shift_reg[29:26] + 3;
            shift_reg = shift_reg << 1;
        end
        ones      = shift_reg[17:14];
        tens      = shift_reg[21:18];
        hundreds  = shift_reg[25:22];
        thousands = shift_reg[29:26];
    end
endmodule
```
