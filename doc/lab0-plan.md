# Lab0 Implementation Plan

## Target

Build a RV64GC core around the existing Nanshan/Chisel skeleton and keep the
verification loop runnable at every stage.

Architectural target:

- ISA: RV64GC, implemented incrementally as IMACFA.
- Privilege: M/S/U modes, CSR, trap/return, timer/software/external interrupt.
- Memory system: AXI4 burst, unblocked L1 cache, TLB and page-table walk.
- Prediction: tournament branch predictor.
- Devices: CLINT, PLIC and UART.
- Verification: function model plus performance model.
- Tests: riscv-tests, cpu-tests, CoreMark, Dhrystone, microbench, RT-Thread,
  nommu-Linux and Linux.
- Flows: Verilator, VCS, DC and FPGA.

## Milestones

1. Bring-up baseline

- Keep a one-instruction commit path connected to difftest.
- Add hardware counters for cycle, commit, stalls, cache/TLB misses and branch
  prediction events.
- Pass basic RV64I arithmetic, load/store and branch tests before adding
  pipeline depth.

2. In-order pipeline

- Split IF/ID/EX/MEM/WB with valid-ready handshakes.
- Add bypassing, load-use interlock, flush and precise commit.
- Make difftest observe only retired instructions.

3. RV64IMAC

- Expand the rvdecoderdb decode table into an internal micro-op format.
- Implement integer ALU, branch/jump, load/store, M extension and A extension.
- Add compressed instruction predecode and correct `isRVC` commit reporting.

4. Privilege and CSR

- Implement M/S/U privilege, trap entry/return, delegation and interrupt priority.
- Wire CLINT timer/software interrupt and PLIC external interrupt.
- Pass privileged riscv-tests before enabling Linux work.

5. AXI4 and cache

- Replace the RAM-only core port with AXI4 master ports.
- Add an unblocked I-cache/D-cache path with MSHR or miss-status tracking.
- Add cache performance events and memory ordering rules required by A/FENCE.

6. MMU

- Add ITLB/DTLB, Sv39 page-table walk, permission checks and fault reporting.
- Integrate TLB shootdown/fence handling.
- Move from nommu-Linux to Linux after page faults and interrupts are stable.

7. BPU and performance

- Add BTB, RAS and tournament direction predictor.
- Measure CoreMark IPC through hardware counters.
- Optimize top stalls first: frontend bubbles, load-use hazards, cache miss
  service time and branch mispredict recovery.

8. Physical and FPGA closure

- Keep synthesis-friendly Chisel patterns and avoid simulation-only paths in
  the synthesizable top.
- Add clock/reset constraints and timing reports for 100 MHz FPGA/DC targets.

## Near-Term Rules

- Do not grow ISA support without a corresponding directed test or difftest
  check.
- Every pipeline stage must carry `valid`, `pc`, `inst`, exception metadata and
  a decoded micro-op.
- All performance optimizations must first define a counter so the bottleneck is
  visible in CoreMark and microbench.
- Keep the simulation top (`SimTop`) stable for difftest; board/SoC integration
  should wrap the core instead of replacing the simulation interface.
