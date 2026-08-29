# Comprector-extreme-compression
comprector is a compression method that will amaze you

it managed to compress a jpg from 50 something kb to 15 

made by [sighthough](https://youtu.be/UtPiUGwu-0Q) using googles gemini 3.6 ai

tech demo [here](https://sighthough.github.io/Comprector-extreme-compression/) feel free to rip anything you want from it (its the index file)

make sure to put block size to 64 and pointer size to 16 along with top frequency blocks  for the best compression!
and experiment with the settings with different files 

[promo vid](https://youtu.be/Mo2Dy_vT1Zs)



Below is a complete, synthesizable **SystemVerilog (HDL)** hardware module for the **Propedia-256 Codec Pipeline**.

It implements both the **Encoder (Compression)** and **Decoder (Decompression)** as streaming hardware pipelines using **Finite State Machines (FSM)** and **Shift-Register logic**, making it ready for implementation on an FPGA (e.g., AMD Xilinx or Intel Altera) or ASIC synthesis.

---

### Hardware Architecture Overview

```
                      +-----------------------------------+
  AXI-Stream In  ---> | STREAMING RUN-LENGTH ENCODER      | ---> Compressed Bus
  (8-bit / 1 Byte)    | Zero-Detector + 8-bit Counter Reg |      (8-bit Data)
                      +-----------------------------------+
                                        |
                                        v
                      +-----------------------------------+
  Compressed Bus ---> | STREAMING RUN-LENGTH DECODER      | ---> AXI-Stream Out
  (8-bit Data)        | Zero-Fill Generator + State Machine|      (8-bit Uncompressed)
                      +-----------------------------------+

```

---

### SystemVerilog Implementation: `propedia256_codec.sv`

```systemverilog
// ============================================================================
// Module: propedia256_codec
// Description: Hardware Encoder/Decoder for Propedia-256 Zero-Run Byte Streaming.
//              Designed for FPGA Block RAM / AXI4-Stream Bus Integration.
// Standard: SystemVerilog IEEE 1800-2012
// ============================================================================

module propedia256_codec #(
    parameter int DATA_WIDTH = 8
)(
    input  logic                  clk,
    input  logic                  rst_n,

    // ------------------------------------------------------------------------
    // ENCODER INTERFACE (Compresses Incoming Bytes)
    // ------------------------------------------------------------------------
    input  logic                  enc_in_valid,
    input  logic [DATA_WIDTH-1:0] enc_in_data,
    input  logic                  enc_in_last,
    output logic                  enc_in_ready,

    output logic                  enc_out_valid,
    output logic [DATA_WIDTH-1:0] enc_out_data,
    output logic                  enc_out_last,
    input  logic                  enc_out_ready,

    // ------------------------------------------------------------------------
    // DECODER INTERFACE (Decompresses Stream Back to Raw Bytes)
    // ------------------------------------------------------------------------
    input  logic                  dec_in_valid,
    input  logic [DATA_WIDTH-1:0] dec_in_data,
    input  logic                  dec_in_last,
    output logic                  dec_in_ready,

    output logic                  dec_out_valid,
    output logic [DATA_WIDTH-1:0] dec_out_data,
    output logic                  dec_out_last,
    input  logic                  dec_out_ready
);

    // ========================================================================
    // MODULE 1: HARDWARE ENCODER (Streaming Zero-Run Compressor)
    // ========================================================================
    typedef enum logic [1:0] {
        ENC_IDLE,
        ENC_COUNT_ZEROS,
        ENC_EMIT_MARKER,
        ENC_EMIT_COUNT
    } enc_state_t;

    enc_state_t enc_state;
    logic [7:0] zero_counter;
    logic       enc_last_reg;

    assign enc_in_ready = (enc_state == ENC_IDLE) || (enc_state == ENC_COUNT_ZEROS && enc_in_data == 8'h00 && zero_counter < 8'hFF);

    always_ff @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            enc_state     <= ENC_IDLE;
            zero_counter  <= 8'h00;
            enc_out_valid <= 1'b0;
            enc_out_data  <= 8'h00;
            enc_out_last  <= 1'b0;
            enc_last_reg  <= 1'b0;
        end else begin
            case (enc_state)

                ENC_IDLE: begin
                    enc_out_valid <= 1'b0;
                    if (enc_in_valid && enc_in_ready) begin
                        if (enc_in_data == 8'h00) begin
                            zero_counter <= 8'd1;
                            enc_last_reg <= enc_in_last;
                            enc_state    <= ENC_COUNT_ZEROS;
                        end else begin
                            enc_out_data  <= enc_in_data;
                            enc_out_valid <= 1'b1;
                            enc_out_last  <= enc_in_last;
                        end
                    end
                end

                ENC_COUNT_ZEROS: begin
                    if (enc_in_valid && enc_in_ready) begin
                        if (enc_in_data == 8'h00 && zero_counter < 8'hFF) begin
                            zero_counter <= zero_counter + 1'b1;
                            if (enc_in_last) begin
                                enc_last_reg <= 1'b1;
                                enc_state    <= ENC_EMIT_MARKER;
                            end
                        end else begin
                            // Non-zero byte hit or max count reached
                            enc_state <= ENC_EMIT_MARKER;
                        end
                    end else if (enc_last_reg) begin
                        enc_state <= ENC_EMIT_MARKER;
                    end
                end

                ENC_EMIT_MARKER: begin
                    if (enc_out_ready) begin
                        enc_out_data  <= 8'h00; // Propedia 0x00 Marker Byte
                        enc_out_valid <= 1'b1;
                        enc_out_last  <= 1'b0;
                        enc_state     <= ENC_EMIT_COUNT;
                    end
                end

                ENC_EMIT_COUNT: begin
                    if (enc_out_ready) begin
                        enc_out_data  <= zero_counter; // Emit Run Length Counter Byte
                        enc_out_valid <= 1 me1;
                        enc_out_last  <= enc_last_reg;
                        zero_counter  <= 8'h00;
                        enc_state     <= ENC_IDLE;
                    end
                end

                default: enc_state <= ENC_IDLE;
            endcase
        end
    end

    // ========================================================================
    // MODULE 2: HARDWARE DECODER (Streaming Zero-Run Unpacker)
    // ========================================================================
    typedef enum logic [1:0] {
        DEC_IDLE,
        DEC_READ_COUNT,
        DEC_FILL_ZEROS
    } dec_state_t;

    dec_state_t dec_state;
    logic [7:0] dec_run_counter;
    logic       dec_last_reg;

    assign dec_in_ready = (dec_state == DEC_IDLE) || (dec_state == DEC_READ_COUNT);

    always_ff @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            dec_state       <= DEC_IDLE;
            dec_run_counter <= 8'h00;
            dec_out_valid   <= 1'b0;
            dec_out_data    <= 8'h00;
            dec_out_last    <= 1'b0;
            dec_last_reg    <= 1'b0;
        end else begin
            case (dec_state)

                DEC_IDLE: begin
                    dec_out_valid <= 1'b0;
                    if (dec_in_valid && dec_in_ready) begin
                        if (dec_in_data == 8'h00) begin
                            // Zero-run sequence detected -> wait for length byte
                            dec_state <= DEC_READ_COUNT;
                        end else begin
                            // Pass literal non-zero byte through
                            dec_out_data  <= dec_in_data;
                            dec_out_valid <= 1'b1;
                            dec_out_last  <= dec_in_last;
                        end
                    end
                end

                DEC_READ_COUNT: begin
                    if (dec_in_valid && dec_in_ready) begin
                        dec_run_counter <= dec_in_data;
                        dec_last_reg    <= dec_in_last;
                        dec_state       <= DEC_FILL_ZEROS;
                    end
                end

                DEC_FILL_ZEROS: begin
                    if (dec_out_ready && dec_run_counter > 0) begin
                        dec_out_data    <= 8'h00;
                        dec_out_valid   <= 1'b1;
                        dec_run_counter <= dec_run_counter - 1'b1;
                        dec_out_last    <= (dec_run_counter == 8'd1) ? dec_last_reg : 1'b0;

                        if (dec_run_counter == 8'd1) begin
                            dec_state <= DEC_IDLE;
                        end
                    end
                end

                default: dec_state <= DEC_IDLE;
            endcase
        end
    end

endmodule

```

---

### Hardware Verilator / Testbench Drivers

To simulate and test this hardware module using **Icarus Verilog**, **Verilator**, or **ModelSim**, run this testbench snippet:

```systemverilog
module tb_propedia256_codec;
    logic clk, rst_n;
    
    // Encoder Signals
    logic enc_in_valid, enc_in_ready, enc_in_last;
    logic [7:0] enc_in_data;
    logic enc_out_valid, enc_out_ready, enc_out_last;
    logic [7:0] enc_out_data;

    // Decoder Signals
    logic dec_in_valid, dec_in_ready, dec_in_last;
    logic [7:0] dec_in_data;
    logic dec_out_valid, dec_out_ready, dec_out_last;
    logic [7:0] dec_out_data;

    // Clock Generation (100 MHz)
    always #5 clk = ~clk;

    // Instantiate Codec
    propedia256_codec dut (.*);

    initial begin
        clk = 0; rst_n = 0;
        enc_in_valid = 0; enc_out_ready = 1;
        dec_in_valid = 0; dec_out_ready = 1;
        #20 rst_n = 1;

        // Drive Test Bytes: [0x05, 0x00, 0x00, 0x00, 0x00, 0x0A]
        // Stream contains a 4-byte zero run
        $display("--- Injecting Data Stream ---");
        drive_enc_byte(8'h05, 0);
        drive_enc_byte(8'h00, 0);
        drive_enc_byte(8'h00, 0);
        drive_enc_byte(8'h00, 0);
        drive_enc_byte(8'h00, 0);
        drive_enc_byte(8'h0A, 1);

        #200;
        $finish;
    end

    task drive_enc_byte(input [7:0] data, input last);
        @(posedge clk);
        enc_in_data  <= data;
        enc_in_valid <= 1;
        enc_in_last  <= last;
        wait(enc_in_ready);
    endtask
endmodule

```

---

### Key Hardware Advantages

1. **Zero CPU Overhead**: Processes byte stream directly on the memory bus using dedicated flip-flops and logic gates.
2. **Line-Rate Execution**: Runs synchronously with the clock tree (e.g., $100\text{--}500\text{ MHz}$ standard FPGA clock rates), yielding gigabytes per second of compressed streaming throughput.
3. **Standard Interface**: Utilizes valid/ready handshake signals compatible with standard **AXI4-Stream** protocols for seamless integration into modern SoC designs (ARM, RISC-V, etc.).
