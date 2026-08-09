# 8-Bit Computer in Logisim Evolution

This is a full 8-bit computer, built gate by gate in Logisim Evolution. No CPU IP block, no pre-built ALU component — every register, every bus tap, every line of microcode was wired by hand and checked against how the real thing behaves. It follows the classic Ben Eater / SAP-1 style: one shared 8-bit bus, a handful of registers sitting on that bus, and a control unit deciding who gets to use it.

There's also a [Verilog port of this exact design](https://github.com/Ramakrishna-MN/8-Bit-Cpu-Verilog), built to run on a Zynq board with real switches and a real 7-segment display instead of a simulator. Both repos describe the same machine.

<p align="center">
  <a href="https://github.com/Ramakrishna-MN/8-bit-computer-logisim/blob/main/Screenshots/CPU.png">
    <img src="https://github.com/Ramakrishna-MN/8-bit-computer-logisim/raw/main/Screenshots/CPU.png" alt="Full CPU schematic in Logisim">
  </a>
</p>

## The core idea: one bus, everyone shares it

Every block in this computer — the program counter, the memory address register, RAM, the instruction register, both general registers, the ALU, the output register — sits on **one shared bus**. Nothing has a private wire to anything else.

When a register wants to hand data to another register, it goes onto the bus through a tri-state buffer. Whoever needs that data reads it off the bus at the same moment. Only one thing is allowed to drive the bus at a time, and the control unit's entire job is making sure that rule never breaks.

That's the whole trick behind why this computer only needs 17 control signals instead of a maze of dedicated wiring. Once the bus makes sense, the rest of the schematic reads itself.

A few things about this particular build are easy to miss just from looking at the schematic, so here they are up front:

**The instruction register only puts 4 bits back on the bus.**
It's an 8-bit register — it has to be, since it stores a full opcode-plus-operand byte when an instruction gets fetched. But when it drives the bus, only the bottom 4 bits (the operand nibble) actually go out. The top 4 bits get pulled to zero. By the time the operand is needed on the bus, the opcode nibble has already done its job, so there's no reason to push it back out.

**The memory address register and program counter are only 4 bits wide.**
That tracks — RAM only has 16 addressable slots. When either one drives the bus, the upper 4 bits are forced to zero rather than left floating. And when either one latches a value *from* the bus, only the bottom 4 bits stick — anything in the upper nibble gets silently dropped. Worth remembering if a jump or memory access ever lands somewhere unexpected.

**The opcode never actually touches the bus.**
The instruction register's top 4 bits go straight into the control unit instead, feeding the low bits of the control ROM's address. The remaining 3 address bits come from the T-state counter. So the control ROM is addressed as `{opcode[3:0], tstate[2:0]}` — 7 bits, 128 words — and that's the entire mechanism deciding what the machine does on any given clock edge.

**RAM and the memory address register run on a faster clock than everything else.**
Every other register — PC, IR, both general registers, the output register, the T-state counter — updates on the slow clock edge. RAM and the MAR tick at double that rate. The reasoning: whatever the control unit decided last cycle needs to be latched into RAM or the MAR *before* the rest of the system's next edge arrives expecting that data to already be valid. If you ever rebuild this for an FPGA, this part can be made negative-edge triggered instead — the values just still need to be picked up from the bus on the positive edge, since that's the assumption the rest of the design leans on.

## The 17 control lines

Every micro-op in this machine comes down to one of 17 signals turning on. Here's the full map:

| Bit | Signal | What it does |
|-----|--------|---------------|
| 0 | OI | Latch the bus into the output register |
| 1 | SU | ALU op select — 0 = add, 1 = subtract |
| 2 | EO | Drive the ALU's result onto the bus |
| 3 | AI | Latch the bus into register A |
| 4 | AO | Drive register A onto the bus |
| 5 | BI | Latch the bus into register B |
| 6 | BO | Drive register B onto the bus |
| 7 | HLT | Halt — freeze the clock for every unit |
| 8 | IO | Drive IR[3:0] onto the bus |
| 9 | II | Latch the full 8-bit bus into the IR |
| 10 | MO | Drive the MAR onto the bus |
| 11 | MI | Latch the bus into the MAR |
| 12 | RO | Drive RAM[MAR] onto the bus |
| 13 | RI | Latch the bus into RAM[MAR] |
| 14 | J | Latch the bus into the PC (jump) |
| 15 | CE | Increment the PC |
| 16 | CO | Drive the PC onto the bus |

Bit 16 is the most significant bit of the control word, bit 0 is the least significant.

### Decoding one, so you can decode the rest yourself

Take the value `10800`. Spread across all 17 bits, that's:

```
1 0 0 0 0 1 0 0 0 0 0 0 0 0 0 0 0
```

Match that against the table above and only two bits are set: bit 16 (**CO**) and bit 11 (**MI**). Everything else sits idle.

So `10800` just means: *push the program counter onto the bus, and latch that value into the memory address register.* That's the very first thing every single instruction does — before RAM can be read, the MAR needs to know which address to look at, and that address always starts life in the PC. Once you can read one control word this way, you can read all of them.

## Instruction set

| Mnemonic | Opcode | Byte layout | Behavior |
|----------|--------|-------------|----------|
| `LDA n`  | `0000` | `0000 nnnn` | A ← RAM[n] |
| `ADD n`  | `0001` | `0001 nnnn` | B ← RAM[n], then A ← A + B |
| `SUB n`  | `0010` | `0010 nnnn` | B ← RAM[n], then A ← A − B |
| `STA n`  | `0100` | `0100 nnnn` | RAM[n] ← A |
| `LDI v`  | `0101` | `0101 vvvv` | A ← v (immediate) |
| `JUMP n` | `1100` | `1100 nnnn` | PC ← n |
| `OUT`    | `1110` | `1110 0000` | Display ← A |
| `HLT`    | `1111` | `1111 0000` | Stop the clock |

No flags register anywhere in this design, so no conditional jumps either. That's not an oversight — the control ROM's 7-bit address (`opcode + tstate`) simply has no spare bits to branch on a carry or zero flag. More on that below, under Roadmap.

## Requirements

- Logisim Evolution, version 3.9.0 or newer
- A Java Runtime Environment (Logisim Evolution runs on the JVM)
- Git, if you'd rather clone than download a zip

## Getting it running

```
git clone https://github.com/Ramakrishna-MN/8-bit-computer-logisim.git
cd 8-bit-computer-logisim
```

Open `8_Bit_CPU.circ` **from inside the Logisim Evolution app**, not by double-clicking it through a file browser — opening it any other way tends to cause rendering glitches with the custom components.

The RAM ships empty. There's no baked-in program — you load one in by hand, the same way you'd toggle switches on a real vintage machine, covered below.

<p align="center">
  <a href="https://github.com/Ramakrishna-MN/8-bit-computer-logisim/blob/main/Screenshots/Output%20display%20in%20decimal.png">
    <img src="https://github.com/Ramakrishna-MN/8-bit-computer-logisim/raw/main/Screenshots/Output%20display%20in%20decimal.png" alt="Output display RAM modules">
  </a>
</p>

<p align="center">
  <a href="https://github.com/Ramakrishna-MN/8-bit-computer-logisim/blob/main/Screenshots/Control_unit%20of%20cpu.png">
    <img src="https://github.com/Ramakrishna-MN/8-bit-computer-logisim/raw/main/Screenshots/Control_unit%20of%20cpu.png" alt="Control unit ROM">
  </a>
</p>

## Programming the computer by hand

There's no assembler, no loader script — you program this the way you'd program an actual breadboard computer, by toggling probes and pulsing the clock one byte at a time.

**Step 1 — flip into programming mode.**
Set the Program Enable probe high. Instructions now go into RAM instead of being executed.

**Step 2 — pick an address.**
The 4-bit Location Probe selects where the next byte lands, anywhere from `0000` to `1111`.

**Step 3 — set the instruction.**
The 8-bit Instruction Probe holds the byte you're writing — top 4 bits are the opcode, bottom 4 are the operand (an address, or an immediate value for `LDI`). For example, `LDI 5` is `01010101`.

**Step 4 — write it.**
Tick the clock once — a full high-then-low pulse — and that byte lands in RAM at the address you picked.

**Step 5 — repeat.**
Move the Location Probe to the next address, set the next instruction, tick the clock again. Keep going until the whole program (and any raw data it needs) is loaded.

Here's a small example — add 5 and 10, then display the result:

| Address | Instruction | Binary   | What it does |
|---------|-------------|----------|---------------|
| 0000    | LDI 5       | 01010101 | Load 5 into A |
| 0001    | ADD 15      | 00011110 | Add the value at address 15 |
| 0010    | OUT         | 11100000 | Push A to the display |
| 0011    | HLT         | 11110000 | Halt |
| 1110    | (raw data)  | 00001010 | The value 10, stored ahead of time for ADD to read |

<p align="center">
  <a href="https://github.com/Ramakrishna-MN/8-bit-computer-logisim/blob/main/Screenshots/Sample%20program.png">
    <img src="https://github.com/Ramakrishna-MN/8-bit-computer-logisim/raw/main/Screenshots/Sample%20program.png" alt="Sample program loaded in hexadecimal">
  </a>
</p>

**Step 6 — run it.**
Flip Program Enable back low — the machine switches from programming mode to execution mode. Start the clock and either run it continuously (Ctrl+K in Logisim) or step through one tick at a time and watch each micro-op fire. Keep an eye on register A and the display; you'll see the sum build up live as fetch, decode, and execute cycle through.

Run the program above and it should land on 15.

<p align="center">
  <a href="https://github.com/Ramakrishna-MN/8-bit-computer-logisim/blob/main/Screenshots/Sample%20output.png">
    <img src="https://github.com/Ramakrishna-MN/8-bit-computer-logisim/raw/main/Screenshots/Sample%20output.png" alt="Sample output on the seven segment display">
  </a>
</p>

That's the whole loop: write assembly in your head, hand-translate it into opcodes and operands, toggle it into RAM one byte at a time, and watch the microcode step through fetching, decoding, and executing it — exactly the way a much older, much slower computer would have.

## Roadmap

**A flags register and conditional jumps.**
The ALU's carry and zero outputs currently go nowhere — nothing captures them. Adding a small flags register and widening the control ROM's address by 2 bits (7 → 9, taking it to 512 words) would open the door to `JC`/`JZ`-style instructions, gated on those flag bits, decoded the same way `JUMP` already is.

**A bigger instruction set.**
Only 8 of the 16 possible opcodes are used right now. `NOP`, flag-conditioned jumps, an `OUT B` for the second register, and maybe a logical op like `AND` or `OR` are all reasonable additions once there's room for them.

**An assembler.**
Hand-toggling bytes in one at a time is a great way to actually understand what the computer is doing, but it stops being fun past about ten instructions. A small tool that turns assembly text into a probe sequence, or a preloadable RAM image, is on the list.

**Keeping this repo in sync with the Verilog port.**
The Verilog version was built by decoding this circuit's ROM contents directly, and a couple of small corrections came out of that (the `LDI` control word, in particular). Those fixes are getting folded back into this `.circ` so both repos keep describing the exact same machine.

## About

This was built to see, at the gate level, what actually happens between "write a program" and "the computer runs it" — not as a black box, but as a specific sequence of specific wires turning on and off. Meant to be opened up, poked at, and watched ticking through its own logic one clock edge at a time.

## License

MIT — use it, modify it, teach with it, break it and fix it again.
