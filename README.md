# DES in Verilog — Design Writeup

**This is an AI generated markdown file to explain my coursework. ALL ACTUAL
COURSEWORK IS MY OWN - no use of AI. Coursework completed in December 2025.**

Three hardware implementations of the DES block cipher sharing one round datapath: a **combinational** design, an **iterative** design, and a **pipelined** design. Written in Verilog, simulated and verified under Icarus Verilog against the supplied test vectors. University coursework, December 2025.

This repository is the writeup only. The implementation is assessed coursework and stays private — see [Availability](#availability).

---

## What I wrote

The assignment supplied a skeleton, block diagrams showing what connects to what, self-checking testbenches, and `util.v`, which contains the permutation tables (`IP`, `FP`, `E`, `P`, `PC1`, `PC2`) and the eight S-boxes. Those are constants defined by the DES standard, so writing them would have been transcription rather than design.

I wrote `round`, `key_schedule`, `clr_28bit`, and the three top-level designs. Mapping the block diagrams onto RTL, deciding where registers go, and sequencing the iterative design were the actual work.

## The problem

DES encrypts a 64-bit block under a 64-bit key across 16 Feistel rounds. Each round takes the two 32-bit halves of the block plus a 48-bit round key and produces the halves for the next round. A key schedule derives each round key from the previous 56-bit key state by rotating its two 28-bit halves.

That structure fixes the amount of work but says nothing about how it maps onto hardware. Sixteen rounds can be sixteen pieces of logic, one piece used sixteen times, or sixteen pieces with registers between them. Same arithmetic, three very different circuits. Building all three is the point of the exercise.

![Datapath](diagrams/datapath.svg)

## Shared building blocks

All three top-level designs use the same `round` and `key_schedule`, so the differences between them are purely structural.

**`round`** implements one Feistel round. The right half is expanded to 48 bits by permutation `E`, XORed with the round key, split into eight 6-bit groups that each index a different S-box, recombined to 32 bits, permuted by `P`, and XORed with the left half. The halves are then swapped. There is no state here — a round is a pure function of `(xl, xr, k)`, which is what makes it reusable across all three architectures.

**`key_schedule`** splits the 56-bit key state into two 28-bit halves, applies a cyclic left rotation to each by an amount depending on the round index, recombines them, and derives the 48-bit round key with permutation `PC2`. It emits both the round key and the next key state, so it can be chained in space or fed back in time depending on the caller.

**`clr_28bit`** is the rotation. DES rotates by one bit in rounds 0, 1, 8 and 15 and by two elsewhere, so the shift amount is a function of the round index rather than a constant.

## Design 1 — combinational

Sixteen `round` instances and sixteen `key_schedule` instances chained by a `generate` loop, with round 0 instantiated separately because it takes its input from the initial permutation rather than from a predecessor.

No clock and no registers. The ciphertext is a pure combinational function of the inputs and appears once the logic settles. This makes it the simplest design to reason about and the one with the longest critical path by a wide margin — all 16 rounds sit in a single unbroken path from `m` to `c`.

It is most useful as a correctness reference. If the round and key schedule are right, this passes, so any failure in the other two designs is a sequencing problem rather than a datapath problem.

## Design 2 — iterative

One `round` instance and one `key_schedule` instance, with registers holding `xl`, `xr`, the key state `kx`, and a round counter `i`. A four-state FSM sequences them. The FSM structure follows the one given in the unit's lecture material; I implemented and debugged it.

![FSM](diagrams/fsm.svg)

`Wait` sits idle until `req` goes high. `Load` captures the permuted message and key into the registers and zeroes the counter. `Round` feeds the register outputs through the round logic and writes the results back, incrementing `i`, until the counter reaches 15. `Done` asserts `ack` and holds it until the caller drops `req`, at which point the machine returns to `Wait`.

Counting edges from `req` sampled high to `ack` asserted: one cycle in `Wait`, one in `Load`, and sixteen in `Round`, giving **18 cycles per block**. There are no IP or FP states — `perm_IP`, `split_2`, `merge_2` and `perm_FP` are all combinational and sit outside the FSM.

That 18 is the design's latency. Sustained throughput is lower. `Wait` holds until `req` arrives and `Done` holds until `req` drops, so back-to-back blocks in this testbench take about 21 cycles each. The extra three cycles are handshake turnaround and belong to the caller rather than to the FSM. Both figures are real; they measure different things, and the distinction is worth keeping straight.

Two details are easy to get wrong:

*The loop terminates at `i < 15`, not `i < 16`.* When `i` is 14 the registers are being loaded with the inputs to the sixteenth round, and the round logic is combinational, so the sixteenth result is on the wires as soon as those registers settle. Testing `i < 16` runs a seventeenth round. The counter is also 4 bits, so it wraps rather than exceeding 15.

*`ack` is a wire, not a register.* It is assigned `Q == Done` combinationally. Registering it would delay it a cycle relative to the state it reports, which the testbench handshake would observe out of step.

This is the smallest of the three designs and the slowest per message.

## Design 3 — pipelined

The unrolled design with registers inserted between rounds.

![Pipeline timing](diagrams/pipeline-timing.svg)

Structurally this is the combinational design again, but each stage's outputs land in `xl[i]`, `xr[i]` and `kx[i]` rather than feeding the next round directly. Stage *i+1* reads the registered outputs of stage *i*, so each clock edge advances every message one round further along.

There are **17 registered stages**: 16 round stages plus the output register on `merge2out`. So the first ciphertext emerges 17 cycles after its plaintext enters, but the **initiation interval is 1 cycle** — once the pipe is full, seventeen messages are in flight simultaneously and one ciphertext emerges per cycle. This is visible directly in the simulation log: the eight real results appear at slots 17 through 24.

The pipelined design also needs no `req`/`ack` handshake, so unlike the iterative design its sustained rate and its initiation interval are the same number.

That trade is the right one for bulk encryption and the wrong one for a small embedded device encrypting one block occasionally. Because the pipeline registers break the critical path down to a single round, this design would also support a higher clock frequency than the combinational one, though I have no synthesis data to quantify that.

### The bug

During development the pipelined design passed some test vectors and failed others, identically on every run. Deterministic, not intermittent.

The cause was a redundant pre-processing register I had added in the datapath before round 0. The commented-out declaration is still in the source:

```verilog
// reg [31:0] splitL, splitR;
```

My diagnosis at the time was a one-cycle skew: registering the split halves delayed the message path while the key path stayed combinational into round 0, so a message was paired with the wrong round key. Removing the register fixed it and all eight vectors passed.

I should be precise about what I can still demonstrate. The register existed and removing it fixed the failures — that is documented in my marksheet from the time. The skew mechanism is my reading of it, consistent with the code structure but not something I have re-run, since the broken version no longer exists.

What I take from it is that a partial pass is a worse signal than a total failure. A design that fails everything is obviously broken. A design that passes most things looks nearly right, and I spent longer than I should have assuming the remaining failures were small.

## Results

Measured under Icarus Verilog against the 8 supplied test vectors.

| Design | Latency (req→ack) | Sustained cycles/block | Critical path | Round instances |
| --- | --- | --- | --- | --- |
| `encrypt_comb` | combinational | — | 16 rounds | 16 |
| `encrypt_iter` | 18 cycles | ~21 incl. handshake | 1 round | 1 |
| `encrypt_pipe` | 17 cycles | 1 cycle | 1 round | 16 |

The two iterative figures measure different things. 18 is the FSM state count from `req` sampled high to `ack` asserted; ~21 is what this testbench actually sustains, and it would change with a different caller. All three designs produce ciphertext matching all 8 vectors.

The pipeline register count, from the RTL declarations: 16 × (32 + 32 + 56) = 1,920 flip-flops, plus 64 for `merge2out`, so roughly 1,900 to 2,000. There are also `permIPout` (64 bits) and `permPC1out` (56 bits), which are clocked but never read — dead registers a synthesiser would strip.

## What I did not measure

**No area or gate counts.** `iverilog -tsizer` and the SystemVerilog size casts in the generate loops turned out to be mutually exclusive: without `-g2012` the `4'(i)` cast is rejected, and with it the sizer backend cannot start from a `$unit` compilation-unit scope. No synthesis ever completed, so I have no LUT counts, no gate counts, and no maximum clock frequency for any of the three designs.

What I have instead is a structural comparison: the combinational and pipelined designs each instantiate 16 `round` and 16 `key_schedule` modules against the iterative design's one of each, and the pipeline adds roughly 1,900 flip-flops on top. That is an instance count, not an area ratio — the iterative design also carries multiplexers, state registers and FSM control that the unrolled designs do not, so the real area difference is smaller than 16×. I can't say by how much.

## Verification

Each design has a self-checking testbench reading known plaintext, key and ciphertext triples from `vectors_m.txt`, `vectors_k.txt` and `vectors_c.txt`. The unit-level modules (`round`, `key_schedule`, `clr_28bit`) have testbenches taking values as plusargs, which let me check individual permutations against hand-worked examples before assembling anything.

Two quirks of the vector set matter when reading the waveforms.

*Vectors 2 through 5 are identical* — the same key and plaintext repeated four times. In the pipelined design this makes adjacent stages hold indistinguishable values while that run is in flight, which looks like a broken pipeline until you check the vector file. The screenshots below are from the early cycles, where the messages in flight are distinct.

*The pipeline log shows `z` inputs and one `x` output near the end.* The testbench stops feeding after 8 vectors but keeps clocking to flush the pipe, so the last 17 slots are drain cycles.

### Waveforms

![Pipeline stages](waveforms/pipeline-stages.png)

Four pipeline stages at the same instant, each holding a different message. A value entering stage 0 appears in stage 1 on the next edge and stage 2 on the one after. The diagonal is the data advancing one round per cycle, which is the whole argument for this design in one frame.

![Iterative handshake](waveforms/iter-handshake.png)

One full transaction of the iterative design. `Q` steps through Wait, Load and Round, the counter `i` climbs 0 to 15 while `Q` holds at 2, then `Q` reaches Done and `ack` is asserted. Note the machine leaves the round state as `i` hits 15 rather than 16. Note also that `Wait` and `Done` occupy more cycles than the transaction itself requires — that is the testbench pacing `req`, and it is where the gap between 18 and 21 comes from.

![Iterative throughput](waveforms/iter-throughput.png)

The same design zoomed out across several transactions. The gaps between `ack` pulses are what the pipelined version eliminates.

## Revisiting the code

Rereading `encrypt_pipe` in 2026, after the coursework was submitted and marked, I found a reset that did nothing. The reset branch assigned the stage registers, and then the advance logic assigned them again, outside an `else`:

```verilog
always @(posedge clk) begin
  if (rst == 1) begin
    xl[0] <= 0; /* ... */
  end
  xl[0] <= wirexl[0];   // not in an else — executes regardless
end
```

Non-blocking assignment schedules the update rather than performing it, so when two assignments to the same register occur in one pass the later one wins. The reset branch ran and was silently overwritten every cycle, and synthesis would have inferred registers with no reset at all. It passed the testbench regardless, because reset is asserted before any meaningful data is in flight and the pipeline flushes itself within 17 cycles.

The iterative design does not have this problem. There the reset is properly exclusive with the `case` statement, so it always lands in `Wait`.

I am listing this under a separate heading because it is not part of what I did in December 2025. It is the same lesson as the pre-processing bug approached from the other direction: the first time, tests failed on code that was nearly right; here, tests passed on code that was wrong in a way simulation could not surface at all.

## Availability

The implementation is assessed university coursework, so the source is kept in a private repository — both because publishing solutions undermines the assignment for future students, and because the provided skeleton is licensed CC BY-NC-ND, which does not permit distributing modified versions. Happy to walk through the code directly or grant repository access on request.

## Credits

The assignment skeleton, testbenches, permutation and S-box definitions (`util.v`), parameters (`params.h`) and build system are Copyright © 2017 Daniel Page <csdsp@bristol.ac.uk>, distributed under the Creative Commons BY-NC-ND 4.0 licence.

The three top-level designs, the round, the key schedule and the rotation module are my own work.
