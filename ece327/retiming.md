always_ff @(posedge clk) begin
    // Data pipeline
    stage1_data  <= input_data;
    stage2_data  <= stage1_data;

    // Valid pipeline — mirrors data pipeline exactly
    stage1_valid <= input_valid;
    stage2_valid <= stage1_valid;
end
```

The consumer only processes output when `output_valid` is asserted. Bubbles propagate as `valid = 0` and are ignored.

**Critical rule:** The valid tag must be registered through **exactly the same number of pipeline stages** as the data. Any mismatch causes the valid signal to arrive at the wrong time relative to its data.

---

## Lecture 8: Retiming

### Definition

**Retiming** is a formal transformation that moves existing pipeline registers **across combinational operators** to rebalance stage delays. It:
- Does **not** add new registers (or removes exactly one if area reduction is specified)
- Does **not** change the circuit's input-output functional behavior
- **Does** change the critical path delay (the goal)
- **Does** change the latency (number of cycles from input to output)

### Retiming Rules

A register can be moved **backward** through an operator (upstream, toward inputs) or **forward** (downstream, toward outputs), subject to one constraint: **no combinational cycle can be created** (every cycle in the graph must contain at least one register after retiming).

**Moving a register forward through an operator:** The register moves from the operator's input to all of its outputs.

**Moving a register backward through an operator:** The register moves from the operator's output to all of its inputs.
```
Before:          After (moved forward):
[REG]──[OP]──    ──[OP]──[REG]
```

If an operator has multiple inputs and you move a register backward, a register must be placed on **every** input — this can increase register count. Conversely, moving forward through a fan-out node reduces register count.

### Retiming Procedure

1. Draw the circuit as a directed graph: nodes = operators (with delays), edges = wires (with register counts)
2. Identify the critical path — the path with maximum delay between any two register boundaries
3. Move registers to shorten that path, ensuring no path loses all its registers (which would create a combinational loop)
4. Recheck all paths for new critical path
5. Repeat until balanced or target met

### Example

Given circuit with operators A(2ns), B(1ns), C(2ns) in sequence with registers:
```
[REG]──[A: 2ns]──[B: 1ns]──[REG]──[C: 2ns]──[REG]
        Critical path = 3ns          CP = 2ns
```

The left stage (A+B = 3ns) is the bottleneck. Move the middle register backward through B:
```
[REG]──[A: 2ns]──[REG]──[B: 1ns]──[C: 2ns]──[REG]
        CP = 2ns            CP = 3ns
```

Still unbalanced. Move the output register backward through C:
```
[REG]──[A: 2ns]──[REG]──[B: 1ns]──[REG]──[C: 2ns]──[REG]
        CP = 2ns           CP = 1ns          CP = 2ns
```

Now balanced. Critical path = 2ns (was 3ns). Same number of registers, higher frequency.

**Key distinction from pipelining:** Pipelining *adds* registers to reduce critical path. Retiming *moves* existing registers — same register count (or fewer), potentially same improvement in critical path.

---

### Pipelining Circuits with Feedback Loops

A standard pipeline assumes data flows in one direction. Some circuits have **feedback** — the output of a stage feeds back to an earlier stage. The canonical example is a recurrence:

$$y[n] = f(x[n],\ y[n-1])$$

The register on the feedback path enforces a **minimum interval of 1 cycle** between successive computations — you cannot start computing $y[n]$ until $y[n-1]$ is available.

**Problem:** You cannot simply add pipeline stages inside the feedback loop. Adding $k$ registers in the loop means $y[n]$ depends on $y[n-k]$, not $y[n-1]$ — the computation is broken.

**Solution — Interleaving:** If you insert $k$ registers in the feedback loop, you can correctly process $k$ **independent streams** interleaved in round-robin fashion. Stream 0 uses the circuit at cycles 0, k, 2k, ... Stream 1 at cycles 1, k+1, 2k+1, ... Each stream sees its own prior output after exactly $k$ cycles.

This achieves full throughput (one operation per cycle across all streams) while maintaining correctness for each individual stream. The tradeoff is that a single stream's throughput is $1/k$ of the clock frequency — only useful when multiple independent streams are available (common in communications and signal processing).

---

### Throughput Balancing in Streaming Architectures

A **streaming architecture** is a chain of processing modules where data flows continuously from one stage to the next. Each module has a characteristic throughput (operations per unit time).

**Fundamental theorem:** The throughput of the entire chain equals the **minimum throughput of any single stage** — the bottleneck.

$$\text{Throughput}_{system} = \min_i(\text{Throughput}_i)$$

This is the pipe and water analogy made formal: regardless of how wide the other pipes are, flow rate through the system is limited by the narrowest pipe.

**Consequence:** Optimizing any stage faster than the bottleneck yields zero system-level improvement. All optimization effort must target the bottleneck stage.

### Throughput Balancing: Module Replication

If a stage has throughput $T/2$ (half what is needed), instantiate **two copies** of it operating in parallel, alternating inputs between them:
```
         ┌──[Module A]──┐
input ───┤              ├─── output
         └──[Module A]──┘
              (copy)
```

Each copy runs at throughput $T/2$, but together they produce one output every half-period → combined throughput $T$.

**Cost:** 2× area, 2× power for that stage. Justified only when that stage is the bottleneck.

### Throughput Balancing: Multi-Pumping

An alternative to replication when the bottleneck module can operate at a **higher clock frequency** than the rest of the system.

If the system clock is $f$ but a module can run at $2f$, clock that module at $2f$ and feed it two inputs per system cycle. It processes both inputs before the next system clock edge, effectively doubling its throughput without duplicating hardware.
```
System clock:  f    →  module sees 2 ops per system cycle
Module clock:  2f   →  processes 2 ops per system cycle
