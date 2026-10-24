# T80: Zilog SCF/CCF (Q register) fix

This is a small fix to **T80 version 350 (Sorgelig, 2018)**, the T80 used by many MiSTer cores. After the fix, the core passes **all 160 tests of `z80full`, `z80ccf` and `z80memptr`** from Patrik Rak's [z80test 1.2a](https://github.com/raxoft/z80test), on real FPGA hardware.

The header of T80 v350 says:

> Test passed: ZEXDOC, ZEXALL, Z80Full(*), Z80memptr
> (*) Currently only SCF and CCF instructions aren't passed X/Y flags check as correct implementation is still unclear.

This patch closes that remaining gap.

## Background

On a Zilog Z80, bits 5 (YF) and 3 (XF) of F after `SCF` or `CCF` depend on whether the **previous instruction modified the flags**. The CPU keeps an internal register, usually called **Q**:

- If the previous instruction wrote F, Q = F.
- Otherwise, Q = 0.

Then the result is:

```
YF/XF after SCF/CCF = (A | (F & ~Q)) & 0x28
```

This was found in 2012 and is documented by David Banks (hoglet) on the [Z80Decoder wiki](https://github.com/hoglet67/Z80Decoder/wiki/Undocumented-Flags#scfccf). `z80ccf` tests it by running a `CCF` after every one of the 160 tested instructions.

## What was wrong in v350

v350 already had a `Q_reg`, but it updated Q only **when F changed value**, using `F /= F_last` at `T_Res`:

1. **Writes with the same value were ignored.** `CP N`, `AND`, `OR`, `BIT` and many other instructions often write F with the value it already had. On the Z80 that still counts as "modified", so Q must become F.
2. **The decision was made at the end of every M-cycle (`T_Res`)**, not once per instruction. Multi-M-cycle instructions such as `RLC (HL)` and `INC (IX+d)` got Q from an intermediate state.
3. **ALU instructions write F late.** T80 writes F_Out through `Save_ALU_r` in the **first cycle of the next instruction**, so the write was credited to the wrong instruction.

## The fix

- A variable `fw` marks every cycle where the current instruction writes F. A signal `F_esc` accumulates it over the whole instruction.
- Q is decided **once per instruction, in its first cycle** (`MCycle = "001"` and `TState = 1`):
  - **Late ALU write in that same cycle** (owned by the previous instruction): Q = F_Out (the value being written).
  - **Else, if the previous instruction wrote F** (`F_esc = '1'`): Q = F.
  - **Otherwise:** Q = 0.
- These do **not** count as F writes, as on a Zilog Z80:
  - `POP AF` and `EX AF,AF'`, which load F rather than compute flags.
  - `SET` and `RES`, which T80 sends through the ALU (`ALU_Op` `1010`/`1011`) and which rewrite F unchanged.
- A `DD`/`FD` prefix behaves as an instruction that does not write F, so Q = 0 before the `SCF`/`CCF` that follows it ([found by TonyB](https://stardot.org.uk/forums/viewtopic.php?p=481595#p481595)).
- **New generic `R800_MULU : boolean := false`.** The R800 `MULUB`/`MULUW` multiplier (MSX turbo R, added in 0250) is not a Zilog Z80 opcode. It is now off by default, which also takes it out of the critical path. Set it to `true` for MSX turbo R cores.

Only `T80.vhd` changes. `T80_ALU.vhd`, `T80_MCode.vhd`, `T80_Reg.vhd`, `T80_Pack.vhd` and `T80se.vhd` are the unmodified v350 files.

## Results

| Test (z80test 1.2a) | Simulation | FPGA hardware |
|---|---|---|
| z80full | pass | **160/160** |
| z80ccf | pass | **160/160** |
| z80memptr | pass (subsets) | **160/160** |

- **Simulation:** GHDL `--synth` → Verilog → Verilator, with `T80se`, `T2Write=1`, a random `CLKEN` and random `WAIT_n`.
- **Hardware:** Gowin GW2AR-18 (Tang Nano 20K), `T80se` at 54 MHz with CLKEN, so the CPU runs at 3.5 and 14 MHz.

## Note on timing when using CLKEN

If you clock T80 fast and slow it down with `CLKEN`, the Fmax report may show T80 paths (IR → F, BusB → Regs) failing at the main clock. Those failures are real on the board: they show up as random `z80full` errors that change between builds.

If CLKEN is never active on two consecutive cycles, tell the tools so with a multicycle constraint. With CLKEN at most once every 3 cycles:

```tcl
set_multicycle_path -from [get_regs {<path>/u0/*}] -to [get_regs {<path>/u0/*}] -setup -end 3
set_multicycle_path -from [get_regs {<path>/u0/*}] -to [get_regs {<path>/u0/*}] -hold  -end 2
```

## Files

- `T80.vhd`: the fixed file (v350 + this patch).
- `T80_Q_fix.patch`: unified diff against the original T80 v350 `T80.vhd`.

## Credits

- **Daniel Wallner**: original T80 (2001–2002, OpenCores).
- **Mike Johnson / MikeJ**: fixes and the T80(b) merge (FPGA Arcade).
- **TobiFlex**: undocumented DDCB/FDCB opcodes, R800 multiplier.
- **Sean Riddle**: 8080 parity fix.
- **Sorgelig**: T80(c) v350 (MiSTer).
- **Patrik Rak (Raxoft)**: z80test.
- **David Banks (hoglet)**: documentation of the SCF/CCF behaviour.
- **TonyB**: found that a DD/FD prefix before SCF/CCF acts as an instruction that does not modify the flags.
- **Rui Ribeiro**: pointed out the SCF/CCF (Q register) issue and its discovery, which led to this fix.

The licence is unchanged: T80's original BSD-style licence, kept in the header of `T80.vhd`.
