# Physical-unclonable-Function
Using Hdl , creating a security system by generating the different keys which are generated using transistors
module puf_cell (
    input  [0:0]  clk_i,
    input  [0:0]  rst_i,
    input  [0:0]  latch_i,
    input  [0:0]  sample_i,
    output [0:0]  data_o
);

    reg [0:0] osc;
    reg [0:0] data_o_reg;

    always @ (*) begin
        if (rst_i) begin
            osc <= 0;
        end
        else if (latch_i == 1) begin
            osc <= !osc;
        end
    end

    always @ (posedge clk_i) begin
        if (sample_i == 1) begin
            data_o_reg <= osc;
        end
    end

    assign data_o[0:0] = data_o_reg[0:0];

endmodule

module top (
    input  [0:0]  clk_i,
    input  [0:0]  rstn_i,
    input  [0:0]  trig_i,
    output [0:0]  busy_o,
    output [95:0] id_o
);

    parameter S_IDLE  = 0,
              S_START = 1,
              S_RUN   = 2,
              S_SAMPL = 3;

    reg [1:0]  arbiter_state;
    reg [96:0] arbiter_sreg;
    reg [0:0]  arbiter_sample;
    reg [0:0]  arbiter_busy;


    // Arbiter sampling FSM.
    always @ (posedge clk_i or negedge rstn_i) begin
        if (!rstn_i) begin
            arbiter_state <= S_IDLE;
        end
        else begin
            case (arbiter_state)
            S_IDLE:
                if (trig_i) begin
                    arbiter_state <= S_START;
                end
            S_START:
                arbiter_state <= S_RUN;
            S_RUN:
                if (arbiter_sreg[96] == 1) begin
                    arbiter_state <= S_SAMPL;
                end
            S_SAMPL:
                arbiter_state <= S_IDLE;
            default:
                arbiter_state <= S_IDLE;
            endcase
        end
    end

    always @ (posedge clk_i) begin
        // Shift left by 1 bit.
        arbiter_sreg[96:1] <= arbiter_sreg[95:0];
        if (arbiter_state == S_START) begin
            arbiter_sreg[0:0] <= 1;
        end
        else begin
            arbiter_sreg[0:0] <= 0;
        end
    end

    always @ (posedge clk_i) begin
        if (arbiter_state == S_SAMPL) begin
            arbiter_sample <= 1;
        end
        else begin
            arbiter_sample <= 0;
        end
 
        if (arbiter_state == S_IDLE) begin
            arbiter_busy <= 0;
        end
        else begin
            arbiter_busy <= 1;
        end
    end

    assign busy_o = arbiter_busy;

    genvar i;
    generate
        for (i = 0; i < 96; i = i + 1) begin
            puf_cell cell0 (
                clk_i,
                arbiter_sreg[i],
                arbiter_sreg[i+1],
                arbiter_sample,
                id_o[i]);
        end
    endgenerate

endmodule
