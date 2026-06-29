# debounce.v

**Module:** `debounce`
**Role:** Button debounce — shared module used on both Board A and Board B
**Board:** Anmaya AT-STLN-ARTIX 7-001 (XC7A35T-FTG256-1)
**Clock:** 24 MHz

## Description

Debounces a mechanical button and outputs a single 1-cycle pulse on the rising edge of the stable button signal.

Uses a 20-bit counter requiring 1,000,000 consecutive cycles of the same stable state before accepting the transition. At 24 MHz this gives a debounce window of approximately 42ms — sufficient to filter contact bounce on all mechanical buttons and switches on the board.

**Two-stage synchroniser** on the raw button input eliminates metastability before the debounce counter sees the signal.

**Outputs a 1-cycle pulse** (`pulse = 1`) only on the rising edge of the stable signal. Falling edges are detected and tracked internally but produce no pulse output. This prevents double-firing and ensures exactly one transaction per button press.

## Ports
| Port | Direction | Width | Description |
|------|-----------|-------|-------------|
| clk | Input | 1 | 24 MHz system clock |
| rst | Input | 1 | Asynchronous reset (active HIGH) |
| btn | Input | 1 | Raw button input from FPGA pin |
| pulse | Output (reg) | 1 | Single 1-cycle pulse on stable rising edge |

```verilog
module debounce (
    input      clk,
    input      rst,
    input      btn,
    output reg pulse
);
    reg [19:0] count;
    reg        btn_sync0, btn_sync1, btn_state;
    always @(posedge clk or posedge rst) begin
        if (rst) begin
            count <= 0; btn_sync0 <= 0; btn_sync1 <= 0; btn_state <= 0; pulse <= 0;
        end else begin
            pulse <= 0;
            btn_sync0 <= btn;
            btn_sync1 <= btn_sync0;
            if (btn_sync1 == btn_state) begin
                count <= 0;
            end else begin
                count <= count + 1;
                if (count == 20'd999999) begin
                    btn_state <= btn_sync1;
                    count <= 0;
                    if (btn_sync1) pulse <= 1;
                end
            end
        end
    end
endmodule
```
