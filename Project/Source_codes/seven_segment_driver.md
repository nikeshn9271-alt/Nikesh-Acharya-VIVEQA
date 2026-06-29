# seven_segment_driver.v

**Module:** `seven_segment_driver`
**Role:** MAX7219 SPI display driver — shared module used on both Board A and Board B
**Board:** Anmaya AT-STLN-ARTIX 7-001 (XC7A35T-FTG256-1)
**Clock:** 24 MHz

## Hardware
- Display IC: MAX7219 (U14)
- Display unit: XLITX 3641AS 4-digit 7-segment
- Level shifters: SN74LVC1T45 (U17/U18/U19) between 3.3V FPGA and 5V MAX7219
- seg_din → J15 (FPGA pin, Bank 15)
- seg_clk → H12 (FPGA pin, Bank 15)
- seg_cs  → J16 (FPGA pin, Bank 15)

## Description

Drives the onboard MAX7219 4-digit 7-segment display via SPI. The MAX7219 requires 16-bit SPI words (8-bit register address + 8-bit data) sent MSB first with CS active LOW.

**Initialisation sequence:** On reset or when show_err/show_blank changes, the driver re-initialises the MAX7219 by sending registers 0–3 (shutdown, decode mode, intensity, scan limit) before sending digit data. needs_reinit flag tracks whether reinitialisation is required.

**SPI clock:** seg_tick_div divides 24 MHz by 24 giving a tick every 24 cycles (~1 MHz SPI clock). Each 16-bit word takes 32 tick edges (16 bits × 2 edges per bit).

**Display modes:**
- Normal: Shows thousands, hundreds, tens, ones digits
- show_err = 1: Shows "Err" across the 3 rightmost digits. Thousands digit blank.
- show_blank = 1: Shuts down the MAX7219 (register 0x0C = 0x00). All digits off.

**Digit order:** The physical digit positions on this board are reversed. Digit register 0x01 drives the leftmost visible digit, 0x04 drives the rightmost.

## Ports
| Port | Direction | Width | Description |
|------|-----------|-------|-------------|
| clk | Input | 1 | 24 MHz system clock |
| rst | Input | 1 | Asynchronous reset |
| thousands | Input | 4 | BCD thousands digit from balance_to_bcd |
| hundreds | Input | 4 | BCD hundreds digit |
| tens | Input | 4 | BCD tens digit |
| ones | Input | 4 | BCD ones digit |
| show_err | Input | 1 | HIGH = display Err (used during freeze on Board A) |
| show_blank | Input | 1 | HIGH = display completely off (Board B standby, Board A shutdown) |
| seg_din | Output (reg) | 1 | SPI data to MAX7219 |
| seg_cs | Output (reg) | 1 | SPI chip select to MAX7219 (active LOW) |
| seg_clk | Output (reg) | 1 | SPI clock to MAX7219 |

```verilog
module seven_segment_driver (
    input        clk,
    input        rst,
    input  [3:0] thousands,
    input  [3:0] hundreds,
    input  [3:0] tens,
    input  [3:0] ones,
    input        show_err,
    input        show_blank,
    output reg   seg_din,
    output reg   seg_cs,
    output reg   seg_clk
);
    reg [4:0] seg_tick_div = 0;
    wire      seg_tick = (seg_tick_div == 23);

    always @(posedge clk or posedge rst) begin
        if (rst) seg_tick_div <= 0;
        else if (seg_tick) seg_tick_div <= 0;
        else seg_tick_div <= seg_tick_div + 1;
    end

    reg prev_show_err   = 0;
    reg prev_show_blank = 0;
    reg needs_reinit    = 1;

    reg [5:0]  seg_state = 0;
    reg [15:0] seg_shift = 0;
    reg [2:0]  curr_dig  = 0;
    reg [15:0] word_to_send;

    always @(*) begin
        case (curr_dig)
            3'd0: word_to_send = 16'h0C01;
            3'd1: begin
                if (show_err || show_blank)
                    word_to_send = 16'h0900;
                else
                    word_to_send = 16'h09FF;
            end
            3'd2: word_to_send = 16'h0A08;
            3'd3: begin
                if (show_blank)
                    word_to_send = 16'h0C00;
                else
                    word_to_send = 16'h0B03;
            end
            // LEFTMOST digit
            3'd4: begin
                if (show_blank)     word_to_send = {8'h01, 8'h00};
                else if (show_err)  word_to_send = {8'h01, 8'h00};
                else                word_to_send = {8'h01, 4'h0, thousands};
            end
            3'd5: begin
                if (show_blank)     word_to_send = {8'h02, 8'h00};
                else if (show_err)  word_to_send = {8'h02, 8'h4F}; // E
                else                word_to_send = {8'h02, 4'h0, hundreds};
            end
            3'd6: begin
                if (show_blank)     word_to_send = {8'h03, 8'h00};
                else if (show_err)  word_to_send = {8'h03, 8'h05}; // r
                else                word_to_send = {8'h03, 4'h0, tens};
            end
            // RIGHTMOST digit
            3'd7: begin
                if (show_blank)     word_to_send = {8'h04, 8'h00};
                else if (show_err)  word_to_send = {8'h04, 8'h05}; // r
                else                word_to_send = {8'h04, 4'h0, ones};
            end
            default: word_to_send = 16'h0000;
        endcase
    end

    always @(posedge clk or posedge rst) begin
        if (rst) begin
            seg_cs <= 1; seg_clk <= 0; seg_din <= 0;
            seg_state <= 0; curr_dig <= 0;
            prev_show_err <= 0; prev_show_blank <= 0; needs_reinit <= 1;
        end else if (seg_tick) begin
            if (show_err != prev_show_err || show_blank != prev_show_blank) begin
                prev_show_err   <= show_err;
                prev_show_blank <= show_blank;
                needs_reinit    <= 1;
            end
            if (seg_state == 0) begin
                seg_cs    <= 0;
                seg_clk   <= 0;
                seg_shift <= word_to_send;
                seg_state <= 1;
            end else if (seg_state <= 32) begin
                if (seg_state[0]) begin
                    seg_din <= seg_shift[15];
                    seg_clk <= 0;
                end else begin
                    seg_clk   <= 1;
                    seg_shift <= {seg_shift[14:0], 1'b0};
                end
                seg_state <= seg_state + 1;
            end else begin
                seg_cs  <= 1;
                seg_clk <= 0;
                if (needs_reinit && curr_dig == 3) begin
                    needs_reinit <= 0;
                    curr_dig     <= 3'd4;
                end else if (curr_dig < 7) begin
                    curr_dig <= curr_dig + 1;
                end else begin
                    curr_dig <= needs_reinit ? 3'd0 : 3'd4;
                end
                seg_state <= 0;
            end
        end
    end
endmodule
```
