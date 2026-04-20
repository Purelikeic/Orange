# Orange Lab0 Codex Guide

本文档给重启后的 Codex/开发者使用。目标是在 `/home/zzj/prj/cpu/Orange`
中从当前简化 Chisel difftest 框架逐步实现 Lab0：

- ISA: RV64GC，按 IMACFA 顺序增量实现。
- Privilege: M/S/U，CSR，异常/中断，`mret/sret`。
- Memory: AXI4 burst，unblocked L1 ICache/DCache，TLB，Sv39 PTW。
- BPU: Tournament predictor，后期用于 CoreMark IPC 优化。
- Devices: CLINT，PLIC，UART。
- Verification: function model + perf model。
- Tests: riscv-tests，cpu-tests，CoreMark，Dhrystone，microbench，
  RT-Thread，nommu-Linux，Linux。
- Flow: Verilator，VCS，DC，FPGA。

## Current Repo Facts

仓库根目录：`/home/zzj/prj/cpu/Orange`

重要文件和目录：

- `build.mill`: Mill/Scala/Chisel 构建入口。`nanshan` 依赖 `rvdecoderdb`
  和 `difftest`。
- `Makefile`: 常用入口。`make verilog` 生成 `build/*.sv`，`make emu`
  会先生成 Verilog 再进入 `difftest` 构建仿真器。
- `nanshan/src`: 当前 CPU RTL 源码。现状是很小的 demo core，不是完整 CPU。
- `nanshan/src/Decode.scala`: 已经接入 `rvdecoderdb` 和 Chisel
  `DecodeTable`，但当前只是示例级 decoder。
- `difftest`: 差分测试框架。文档在 `difftest/doc/usage.md`、
  `difftest/doc/example-nutshell.md`、`difftest/doc/example-xiangshan.md`。
- `nemu`: reference model / function model 候选。
- `rvdecoderdb`: 从 `riscv-opcodes` 解析 RISC-V 指令编码数据库。
- `riscv-opcodes`: 指向 `rvdecoderdb/rvdecoderdbtest/jvm/riscv-opcodes`
  的符号链接。
- `DRAMsim3`: 后期内存性能模型候选，不是当前 CPU RTL 的必要依赖。

当前 RTL 主要能力：

- `InstFetch`: 固定从 `0x80000000` 开始，PC 每周期 `+4`。
- `Decode`: 使用 `rvdecoderdb` 生成少量 decode 字段，但 `Opcode` 目前只
  是 demo。
- `Execution`: 目前只做了类似加法的行为，LSU/BRU/CSR/MUL/DIV 都未成型。
- `Core`: 已接入部分 difftest blackbox，但提交时序还不是完整流水线语义。
- `Ram2r1w`: 使用 `ram_2r1w` blackbox 做简单仿真 RAM。

重启后第一原则：

- 先读 `git status --short`，不要覆盖用户改动。
- 如果用户要求回到起始状态，只还原明确由 Codex 增加/修改的文件。
- 不要一上来实现 RV64GC 全量功能。先做 RV64I + difftest 正确提交闭环。

## Tool Usage

### Mill/Chisel

常用命令：

```bash
cd /home/zzj/prj/cpu/Orange
make verilog
mill -i nanshan.runMain nanshan.Elaborate --target-dir ./build
mill -i __.test
```

`make verilog` 成功后应在 `build/` 看到 `SimTop.sv`、`Core.sv` 等生成文件。

注意：如果出现 `espresso is not found in your PATH`，通常是 Chisel
`DecodeTable` 逻辑最小化工具缺失提示。它可能 fallback 到其他 minimizer，
但应尽早安装，避免 decode 规模变大后生成慢或行为受限制。

推荐安装 Berkeley Espresso logic minimizer：

```bash
sudo apt update
sudo apt install -y build-essential git
mkdir -p ~/Tool
cd ~/Tool
git clone https://github.com/classabbyamp/espresso-logic.git
cd espresso-logic/espresso-src
make
mkdir -p ~/.local/bin
cp bin/espresso ~/.local/bin/espresso
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
command -v espresso
```

不要安装 Ubuntu apt 里的 `quantum-espresso`，那是量子化学软件，不是这里的
logic minimizer。

### rvdecoderdb

`rvdecoderdb` 的职责：从 `riscv-opcodes` 解析指令名、encoding、操作数字段、
所属 extension。它适合生成 decoder pattern，不负责实现执行语义。

当前使用方式在 `nanshan/src/Decode.scala`：

```scala
import org.chipsalliance.rvdecoderdb

case class Insn(val inst: rvdecoderdb.Instruction) extends DecodePattern {
  override def bitPat: BitPat = BitPat("b" + inst.encoding.toString())
}

val instTable = rvdecoderdb.fromFile.instructions(os.pwd / "riscv-opcodes")
```

后续建议：

- 用 `rvdecoderdb.instructions(os.pwd / "riscv-opcodes")` 替代 deprecated
  `rvdecoderdb.fromFile.instructions`。
- 根据 `instructionSet.name` 筛选 extension，例如 `rv_i`、`rv64_i`、
  `rv_m`、`rv64_m`、`rv_a`、`rv_f`、`rv64_f`。
- 根据 `inst.name` 生成执行单元类型、操作类型、访存宽度、写回使能等字段。
- 根据 `inst.args.map(_.name)` 生成 `rs1_en`、`rs2_en`、`rd_en` 和立即数类型。
- 不要只输出一个 8-bit `opcode`。应输出结构化 `MicroOp`。

建议的 decode 输出：

```scala
class MicroOp extends Bundle {
  val valid    = Bool()
  val illegal  = Bool()
  val fuType   = UInt(...)
  val opType   = UInt(...)
  val src1Type = UInt(...)
  val src2Type = UInt(...)
  val immType  = UInt(...)
  val rfWen    = Bool()
  val memRen   = Bool()
  val memWen   = Bool()
  val memWidth = UInt(...)
  val csrCmd   = UInt(...)
}
```

`rvdecoderdb` 不会自动判断 `add` 走 ALU、`ld` 走 LSU、`csrrw` 走 CSR。这些
映射必须在 `DecodeField.genTable` 里由本项目自己定义。

RVC 注意事项：

- 第一阶段不要做 C 扩展。先实现 32-bit RV64I。
- 后续做 C 扩展时，应优先设计 `PreDecode/Decompress`，将 16-bit compressed
  instruction 展开为内部 32-bit 指令或 micro-op，再进入主 decoder。
- 在接入 RVC 后，difftest 的 `isRVC` 必须正确，PC 步长也必须支持 `+2/+4`。

### difftest

核心文档：

- `difftest/doc/usage.md`
- `difftest/doc/example-nutshell.md`
- `difftest/doc/example-xiangshan.md`

核心原则：

- 传给 difftest 的信号必须代表“指令提交时刻其产生的影响恰好生效”。
- 对顺序单发射 CPU，写回级通常就是提交级。
- GPR/CSR 更新时序要和 `DifftestInstrCommit` 对齐，必要时 `RegNext`。

必须先接好的 difftest 模块：

- `DifftestInstrCommit`: 每条提交指令的 `valid/pc/instr/wen/wdata/wdest`。
- `DifftestArchIntRegState`: 32 个整数寄存器。
- `DifftestCSRState`: CSR 状态。早期可以先固定 machine mode，但做 CSR 后必须真实。
- `DifftestArchEvent`: 异常/中断。
- `DifftestTrapEvent`: trap 结束仿真，带 cycle/instr count。

后期可接：

- `DifftestArchFpRegState`: F/D 扩展。
- `DifftestLoadEvent`、`DifftestStoreEvent`、`DifftestAtomicEvent`、
  `DifftestSbufferEvent`、`DifftestPtwEvent`: 访存、原子、PTW 相关验证。

仿真顶层必须保留：

```scala
class SimTop extends Module {
  val io = IO(new Bundle {
    val logCtrl  = new LogCtrlIO
    val perfInfo = new PerfInfoIO
    val uart     = new UARTIO
  })
}
```

`PerfInfoIO.clean/dump` 只是控制信号，性能计数器实现需要 RTL 自己维护。

### NEMU

NEMU 是 function model / reference model 的核心候选。后续要确认：

- NEMU 当前配置是否已启用 riscv64。
- difftest Makefile 是否已经正确找到 NEMU 动态库或 reference。
- 测试 bin 的 load address 是否与当前 `InstFetch` 和 RAM 模型一致。

不要在 CPU 功能不稳定时先追 Linux。顺序应为：

1. small baremetal。
2. riscv-tests。
3. cpu-tests。
4. CoreMark/Dhrystone。
5. RT-Thread。
6. nommu-Linux。
7. Linux + Sv39。

### DRAMsim3

DRAMsim3 用于后期 perf model / memory timing model，不是初期正确性闭环必需项。

建议使用时机：

- 已有 AXI4 master。
- ICache/DCache miss 路径已经稳定。
- 需要评估 memory latency/bandwidth 对 CoreMark 或 microbench 的影响。

不要在 RV64I/difftest 未稳定前接 DRAMsim3。

## Reference: NutShell

NutShell 是本项目最适合的主参考，不是直接复制对象。

参考点：

- 顺序单发射流水线组织。
- frontend/backend/LSU/CSR/MMU/cache 的模块拆分。
- valid-ready 或 ready-valid 风格的流水线传递。
- difftest 在写回/提交级的接法。
- `AXI4RAM` 使用 `RAMHelper` 做仿真内存的方式。
- CSR/trap/interrupt/MMU 的增量实现顺序。

不要直接复制：

- 完整 9 级流水线。
- 完整 cache/TLB/MMU 依赖链。
- 大量 project-specific parameter/config 体系。

当前 Orange 应先长成“简化 NutShell”：

1. 单发射。
2. 顺序提交。
3. 先 blocking LSU/cache。
4. 后续再 unblocked cache 和 tournament BPU。

XiangShan 只作为后期参考：

- 多发射 difftest。
- load/store/atomic event。
- PTW event。
- 更复杂性能优化思路。

## Implementation Roadmap

### Phase 0: Workspace and Build Baseline

目标：确认仓库能生成 Verilog，工具链可用。

步骤：

1. `git status --short`，记录已有改动。
2. `make verilog`，确认 Chisel 编译和 Verilog 生成。
3. 安装 `espresso`，消除 `DecodeTable` warning。
4. 确认 `build/SimTop.sv`、`build/Core.sv`、`build/Decode.sv` 存在。
5. 只在需要时构建 `difftest` emu。

验收：

- `make verilog` 返回成功。
- 没有 Scala/Chisel compile error。

### Phase 1: RV64I Single-Issue Functional Core

目标：先实现 RV64I 最小正确性闭环，暂不追求性能。

必须改造：

- `Decode.scala`: 输出 `MicroOp`，不要继续扩展 demo `Opcode`。
- `Execution.scala`: 拆出 ALU/BRU/LSU 行为。
- `InstFetch.scala`: 支持 `nextPc`，不能固定 `pc + 4`。
- `Core.scala`: 建立真正的提交信号 `commitValid`。
- `Ram.scala`: 确认 load/store mask、对齐、符号扩展行为。

第一批指令：

- U/J/B/I/R: `lui`，`auipc`，`jal`，`jalr`，`beq`，`bne`，`blt`，`bge`，
  `bltu`，`bgeu`。
- ALU immediate: `addi`，`slti`，`sltiu`，`xori`，`ori`，`andi`，
  `slli`，`srli`，`srai`。
- ALU register: `add`，`sub`，`sll`，`slt`，`sltu`，`xor`，`srl`，
  `sra`，`or`，`and`。
- Load/store: `lb`，`lh`，`lw`，`ld`，`lbu`，`lhu`，`lwu`，`sb`，
  `sh`，`sw`，`sd`。

暂缓：

- CSR。
- fence。
- AMO。
- M extension。
- C/F/A。
- 异常精确处理。

验收：

- 小型手写 baremetal 能执行分支、load/store、函数返回。
- difftest `DifftestInstrCommit` 只在真实提交时 valid。
- `x0` 永远为 0。
- load sign/zero extension 正确。

### Phase 2: Five-Stage or Multi-Stage In-Order Pipeline

目标：从 demo datapath 变成可扩展顺序流水线。

建议阶段：

- IF: PC generation, instruction fetch, redirect handling。
- ID: decode, regfile read, immediate generation。
- EX: ALU/BRU address generation。
- MEM: load/store request/response。
- WB: writeback and commit。

每级必须携带：

- `valid`
- `pc`
- `inst`
- `uop`
- exception metadata
- branch redirect metadata

必须实现：

- flush。
- stall。
- bypass。
- load-use interlock。
- branch/jump redirect。
- commit only at WB。

验收：

- RV64I riscv-tests 基本通过。
- difftest 不应在 flush 掉的指令上提交。

### Phase 3: M Extension

目标：实现 RV64M/RV64M word 操作。

指令：

- `mul`，`mulh`，`mulhsu`，`mulhu`，`div`，`divu`，`rem`，`remu`。
- `mulw`，`divw`，`divuw`，`remw`，`remuw`。

建议：

- 乘法可以先组合或多周期。
- 除法建议多周期 FSM。
- 多周期单元必须能 stall pipeline。

验收：

- RV64M riscv-tests 通过。

### Phase 4: CSR and Machine Mode

目标：实现 Zicsr + M mode trap，支撑中断/异常基础。

先做 CSR：

- `mstatus`
- `mtvec`
- `mepc`
- `mcause`
- `mtval`
- `mie`
- `mip`
- `mscratch`
- `cycle`
- `instret`

指令：

- `csrrw`
- `csrrs`
- `csrrc`
- `csrrwi`
- `csrrsi`
- `csrrci`
- `ecall`
- `ebreak`
- `mret`

异常：

- illegal instruction。
- breakpoint。
- environment call。
- load/store/inst address misaligned。
- access fault 可以先简化。

验收：

- privileged riscv-tests 的 machine-mode 子集逐步通过。
- difftest CSRState 反映真实 CSR。

### Phase 5: CLINT, PLIC, UART

目标：外设基础，支撑 RTOS 和系统软件。

顺序：

1. UART MMIO 输出，先支持 putchar。
2. CLINT `mtime/mtimecmp/msip`。
3. timer interrupt。
4. PLIC external interrupt。

验收：

- baremetal UART 输出。
- timer interrupt 测试。
- RT-Thread 前置环境。

### Phase 6: AXI4

目标：替换 core 内部简单 RAM 接口，提供 AXI4 master。

建议：

- 先做简单 blocking AXI master。
- 再支持 burst。
- 仿真内存参考 NutShell `AXI4RAM` + difftest `RAMHelper`。

不要一开始就引入 Rocket-Chip diplomacy，除非明确计划接受其参数体系和依赖。
如果只需要 AXI bundle，可以先自定义简化 AXI4 Bundle，接口稳定后再考虑复用
Rocket-Chip API。

验收：

- I-fetch 和 LSU 均走 AXI。
- Verilator 下能从 AXI RAM 跑 baremetal。

### Phase 7: Cache

目标：L1 ICache/DCache。

顺序：

1. ICache blocking。
2. DCache blocking。
3. store buffer。
4. miss handling。
5. unblocked cache / MSHR。

必须加性能计数：

- ICache miss。
- DCache miss。
- miss latency。
- load-use stall。
- store buffer full stall。

验收：

- cache on/off 下功能一致。
- CoreMark IPC 可解释。

### Phase 8: S/U Mode, TLB, Sv39

目标：支持 nommu-Linux 到 Linux。

顺序：

1. S/U privilege。
2. `sstatus`、`stvec`、`sepc`、`scause`、`stval`、`satp`。
3. delegation: `medeleg/mideleg`。
4. ITLB/DTLB。
5. PTW。
6. page fault/access fault。
7. `sfence.vma`。

验收：

- supervisor privileged tests。
- nommu-Linux。
- Linux boot early log。

### Phase 9: A/F/C Extensions

目标：补齐 RV64GC 的 A/F/C。

建议顺序：

1. A: LR/SC/AMO，先在单核内保证语义。
2. C: decompressor + `isRVC` difftest。
3. F: FPU register file + FP difftest state。

注意：

- A extension 会影响 memory ordering 和 difftest atomic event。
- C extension 会影响 frontend、PC、commit `isRVC`。
- F extension 会引入 rounding mode、fflags、fcsr。

验收：

- RV64A/F/C tests。
- FP difftest 状态正确。

### Phase 10: BPU and CoreMark Performance

目标：CoreMark IPC 0.6+，Freq 100M+。

顺序：

1. 先加性能计数器。
2. BTB。
3. RAS。
4. local/global predictor。
5. tournament chooser。
6. redirect recovery 优化。

必须统计：

- cycle。
- commit。
- IPC。
- frontend stall。
- backend stall。
- branch count。
- mispredict count。
- ICache/DCache/TLB miss。

优化原则：

- 先测再改。
- 优先优化最大 stall 来源。
- 不要在正确性不稳定时追 IPC。

验收：

- CoreMark 可跑完。
- IPC 由硬件计数器和仿真日志交叉确认。
- 100 MHz FPGA/DC timing 不违背功能路径。

## Decoder Implementation Detail

第一版推荐新增/重构这些定义：

- `nanshan/src/Isa.scala`: XLEN、reset vector、枚举常量。
- `nanshan/src/MicroOp.scala`: `MicroOp` bundle。
- `nanshan/src/Decode.scala`: `rvdecoderdb` + `DecodeTable` 产生 `MicroOp`。

枚举建议先用 `object` 常量，不急着复杂化：

```scala
object FuType {
  val none = 0.U(4.W)
  val alu  = 1.U(4.W)
  val bru  = 2.U(4.W)
  val lsu  = 3.U(4.W)
  val csr  = 4.U(4.W)
  val mul  = 5.U(4.W)
  val div  = 6.U(4.W)
}
```

`DecodeField` 示例思路：

```scala
object FuTypeField extends DecodeField[Insn, UInt] {
  override def name = "fu_type"
  override def chiselType = UInt(4.W)
  override def genTable(i: Insn): BitPat = i.inst.name match {
    case "add" | "sub" | "addi" | "and" | "or" | "xor" => BitPat(FuType.alu)
    case "beq" | "bne" | "jal" | "jalr"               => BitPat(FuType.bru)
    case "lb" | "lh" | "lw" | "ld" | "sb" | "sh" |
         "sw" | "sd"                                   => BitPat(FuType.lsu)
    case _                                             => BitPat(FuType.none)
  }
}
```

实际实现时要注意 `BitPat(...)` 的类型构造是否匹配当前 Chisel 版本。必要时用
`BitPat("b....")` 显式写位宽。

立即数生成：

- I: `Cat(Fill(52, inst(31)), inst(31,20))`
- S: `Cat(Fill(52, inst(31)), inst(31,25), inst(11,7))`
- B: `Cat(Fill(51, inst(31)), inst(31), inst(7), inst(30,25), inst(11,8), 0.U)`
- U: `Cat(Fill(32, inst(31)), inst(31,12), Fill(12, 0.U))`
- J: `Cat(Fill(43, inst(31)), inst(31), inst(19,12), inst(20), inst(30,21), 0.U)`

检查当前代码里的符号扩展位数，确保总宽度正好 64 bit。

非法指令：

- `DecodeTable` 默认值要明确。
- 如果 pattern 未匹配，`uop.illegal := true.B`。
- 早期可以 trap 或直接 halt，但最终必须产生 illegal instruction exception。

## Difftest Timing Checklist

提交级需要形成：

```scala
commit.valid
commit.pc
commit.inst
commit.isRVC
commit.rfWen
commit.rfWaddr
commit.rfWdata
commit.skip
commit.scFailed
```

连接原则：

- `DifftestInstrCommit.valid` 只对真实退休指令有效。
- flush、kill、exception before commit 的指令不能提交。
- MMIO 可以 `skip := true.B`，但必须谨慎，不能用 skip 掩盖普通内存错误。
- `DifftestArchIntRegState` 要读取提交后可见的 GPR 状态。
- CSRState 要和 CSR 更新时机对齐。
- TrapEvent 的 `cycleCnt/instrCnt` 必须来自真实 cycle/commit 计数。

如果采用顺序单发射：

- WB 写回当周期提交。
- 如果 RegFile 写入下一拍才读出，则 InstrCommit 可以 `RegNext(commit)`，
  GPR state 直接读下一拍结果。

## Testing Strategy

每加一类功能，就先写/跑最小测试，再扩展测试集。

测试顺序：

1. 手写 baremetal：几条 ALU 和 branch。
2. load/store byte/half/word/dword。
3. call/return。
4. riscv-tests RV64I。
5. RV64M。
6. CSR/privileged。
7. CoreMark。
8. RT-Thread。
9. nommu-Linux。
10. Linux。

每次提交前至少做：

```bash
make verilog
```

当 difftest emu 可用后：

```bash
make emu
cd difftest
./build/emu --help
```

具体 workload 路径需要根据仓库和 NEMU 配置确认，不要假设已经存在。

## Common Pitfalls

- 不要把 decoder 的 `inst.name` 映射写散在多个模块里。应集中在
  `Decode.scala` 生成 `MicroOp`。
- 不要在没有 `valid` 的情况下做流水线。所有阶段必须能 stall/flush。
- 不要让 difftest 看到 speculative instruction。
- 不要把 `pc + 4` 写死。C 扩展、branch、jump、trap 都会改变 next PC。
- 不要先做 unblocked cache。先 blocking 保证正确。
- 不要先做 Linux。Linux 依赖 CSR、S mode、中断、TLB、设备树/外设等大量基础。
- 不要把 Rocket-Chip API 当成无成本依赖。当前仓库没有直接引入 Rocket-Chip。
- 不要忽略 `x0`。写 x0 必须被丢弃，读 x0 必须为 0。
- load/store mask 和 sign extension 最容易错，应单独测试。
- branch compare 要区分 signed/unsigned。
- `sra/srai` 要算术右移，不能用逻辑右移替代。

## Immediate Next Task After Restart

如果用户要求继续实现，从这里开始：

1. 检查工作树：

```bash
cd /home/zzj/prj/cpu/Orange
git status --short
```

2. 如果用户已经还原到起始状态，先只读这些文件：

```bash
sed -n '1,220p' nanshan/src/Decode.scala
sed -n '1,220p' nanshan/src/Core.scala
sed -n '1,220p' nanshan/src/Execution.scala
sed -n '1,180p' nanshan/src/InstFetch.scala
sed -n '1,180p' nanshan/src/Ram.scala
```

3. 先实现 decoder 重构，不碰 AXI/cache/TLB。

建议第一组代码变更：

- 新增 `nanshan/src/Isa.scala`。
- 新增 `nanshan/src/MicroOp.scala`。
- 重写 `nanshan/src/Decode.scala`，用 `rvdecoderdb` 生成结构化 `MicroOp`。
- 更新 `Execution.scala` 支持 RV64I ALU。
- 更新 `InstFetch.scala` 支持 redirect/nextPc。
- 更新 `Core.scala` 形成最小 commit bus。

4. 第一轮验收：

```bash
make verilog
```

5. 如果 build 过，再加最小测试或接 difftest emu。

## Progress Log

2026-04-19:

- 已确认仓库是 Chisel + Mill + difftest + rvdecoderdb 框架。
- 已确认 `difftest/doc` 主要指导验证接入，不是完整 CPU 实现教程。
- 已确认当前 `Decode.scala` 已使用 `rvdecoderdb`，但只作为 demo decoder。
- 后续应从 RV64I decoder/micro-op 重构开始。
