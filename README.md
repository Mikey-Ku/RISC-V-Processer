# Project 1: Multicycle RV32I CPU

A multicycle, unpipelined RV32I processor in SystemVerilog, running on the iceBlinkPico
(Lattice iCE40 UP5K). It implements every base RV32I instruction except `ecall`, `ebreak`,
and the CSR instructions. It uses the provided `memory.sv` for 4 kB of instruction memory,
4 kB of data memory, and the memory-mapped LED and timer peripherals.

## Demo

`programs/blink.s` running on the board: the RGB LED fades through a rainbow about every
3 seconds while the user LED breathes.

Video coming soon.

## How it works

![Datapath of the multicycle RV32I CPU, colored by state](images/cpu_datapath.svg)

Each instruction moves through a five-state FSM in `cpu.sv`, one state per clock cycle:

| State | What happens | Latched at the end |
| --- | --- | --- |
| FETCH | Instruction memory reads the word at `pc` | `instruction` |
| DECODE | Fields are split out, the register file reads `rs1` and `rs2` | `rs1_value`, `rs2_value` |
| EXECUTE | The ALU computes a result, address, or jump target; the branch unit compares | `alu_result` |
| MEMORY | Loads read and stores write at `alu_result` (loads and stores only) | `load_data` |
| WRITEBACK | Pick `rd_value` and `next_pc`, write `rd` | register `rd`, `pc` |

Most instructions take 4 cycles. Loads and stores take 5.

`pc+4` is the address of the next instruction. Each instruction is 4 bytes, so instructions
sit 4 addresses apart. `next_pc` is `pc+4` unless a jump or taken branch uses the ALU's
target instead. `jal` and `jalr` also write `pc+4` to `rd` as the return address: `jal x13`
at `0x1034` saves `0x1038`, and `jalr x14, 0(x13)` later jumps back there.

The CPU updates on the falling edge of the clock and memory on the rising edge. An address
the CPU sets at one falling edge is used by memory half a cycle later, and the data is
ready by the next falling edge, so every memory access fits in one state. The register file
also reads on the falling edge, which is why DECODE exists: it gives the register file one
edge to read `rs1` and `rs2` before EXECUTE uses them.

### Memory map

| Address | Contents |
| --- | --- |
| `0x00000000` – `0x00000FFF` | Data RAM, byte, half-word, and word access |
| `0x00001000` – `0x00001FFF` | Instruction memory, `pc` starts at `0x1000` |
| `0xFFFFFFF4` (`-12(x0)`) | `micros`, microseconds since start |
| `0xFFFFFFF8` (`-8(x0)`) | `millis`, milliseconds since start |
| `0xFFFFFFFC` (`-4(x0)`) | LEDs, `0xLLRRGGBB` PWM duty cycles |

## Programs

- `programs/blink.s`: the board demo. Steps a color counter every 2 ms using `millis` and
  writes the packed color to the LED register.
- `programs/rv32i_test.s`: the provided test program.
- `programs/rv32i_coverage.s`: covers what the provided test skips: the rest of the
  I-type and R-type ALU instructions, all six branches taken and not taken, every load and
  store width in data RAM, `fence`, writes to `x0`, and `jalr` to an odd address.

## Testing

`top_tb.sv` runs both test programs and checks 49 register values, grouped by instruction
class:

```
programs/rv32i_test
U-type
PASS  lui x1, 0xFEDCC + addi x1, x1, 0xA98         fedcba98
PASS  auipc x10, 0x12345 at 0x1028                 12346028
I-type ALU
PASS  srli x2, x1, 4                               0fedcba9
PASS  srai x3, x1, 4                               ffedcba9
...
ALL TESTS PASSED
```

The design synthesizes for the UP5K at about 18 MHz, above the board's 12 MHz clock.

## Files

- `cpu.sv`: the five-state FSM and datapath
- `register_file.sv`, `immediate.sv`, `alu.sv`, `branch.sv`: CPU submodules
- `memory.sv`, `split_memhfile.py`: provided memory module and hex splitter
- `top.sv`, `iceBlinkPico.pcf`: board top level and pin map
- `top_tb.sv`: testbench
- `programs/`: assembly sources and their memory files
- `images/`: README figures

## Commands

```
make test
make sim
make asm p=blink
make build
make prog
```
