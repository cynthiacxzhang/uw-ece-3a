**FSM Template in SystemVerilog**

```
module fsm (
    input  logic clk, rst, in,
    output logic out
);

// 1. State encoding
typedef enum logic [1:0] {
    IDLE = 2'b00,
    S1   = 2'b01,
    S2   = 2'b10
} state_t;

state_t state, next_state;

// 2. State register — sequential
always_ff @(posedge clk) begin
    if (rst) state <= IDLE;
    else     state <= next_state;
end

// 3. Next-state decoder — combinational
always_comb begin
    case (state)
        IDLE: next_state = in ? S1   : IDLE;
        S1:   next_state = in ? S2   : IDLE;
        S2:   next_state = in ? S2   : IDLE;
        default: next_state = IDLE;
    endcase
end

// 4. Output decoder — combinational (Moore)
always_comb begin
    case (state)
        IDLE: out = 1'b0;
        S1:   out = 1'b0;
        S2:   out = 1'b1;
        default: out = 1'b0;
    endcase
end

endmodule
```

**Chess Clock - Multiple Transitions per FSM State**

The next-state case block must handle all input combinations for each state. With N possible inputs per state, you need to either use nested if-else inside each case arm, or enumerate all input combinations.

```
always_comb begin
    case (state)
        PLAYER_A: begin
            if      (pause)  next_state = PAUSED_A;
            else if (switch) next_state = PLAYER_B;
            else             next_state = PLAYER_A;
        end
        // ...
    endcase
end
```

**Music Toy - Multiple Interacting FSMs**

When a system has independent sub-behaviors, implementing it as a single monolithic FSM causes state explosion — the number of states is the Cartesian product of all sub-behavior state counts.

```
// FSM A: controls melody
// FSM B: controls tempo/rhythm
// They share: tempo_signal, note_signal
// Each implemented as an independent always_ff + always_comb pair
```

*Key lesson:* When two aspects of behavior are orthogonal (neither affects the other's state count), parallel FSMs are strictly better — fewer states, simpler transitions, easier to reason about. This is the same principle used in Practice Q2 from the midterm review (splitting X-output FSM and Y-output FSM).

---

**Polynomial Evaluation- FSM + Datapath (FSMD)**

*FSMD* = Finite State Machine + Datapath. Used when:
- The computation involves a datapath (arithmetic operations on multi-bit values)
- The sequence of operations depends on control decisions (the FSM)

*Example:* Evaluate polynomial `y = ax² + bx + c` on a single multiply-add ALU (one multiply and one add per cycle).

The computation requires multiple cycles because only one multiply-add is available:
1. Cycle 1: compute `a * x` → store in register
2. Cycle 2: compute `(a*x + b) * x` → store
3. Cycle 3: compute result `+ c`

The *FSM* sequences through these steps, asserting control signals (MUX selects, register enables) each cycle. The **datapath** contains the ALU, registers, and MUXes that perform the actual arithmetic.
```
         Control signals
FSM ──────────────────────→ Datapath (ALU, regs, MUXes)
 ↑                                  │
 └──────── Status signals ──────────┘
           (e.g., done, overflow)
```
---

**SystemVerilog Structure**

```
// Datapath: always_comb for ALU, always_ff for result registers
// FSM: always_ff for state, always_comb for next-state and control signals

always_comb begin
    case (state)
        STEP1: begin
            alu_op     = MUL;
            mux_sel    = A_INPUT;
            reg_enable = 1'b1;
            next_state = STEP2;
        end
        // ...
    endcase
end
```
