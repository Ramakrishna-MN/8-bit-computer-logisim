# 8-Bit Computer in Logisim Evolution

This is a full 8-bit computer, designed from the gate level up in Logisim Evolution. There's no CPU IP block, no pre-built ALU component, no shortcuts — every register, every bus tap, every microcode line was wired by hand and then verified bit by bit against how the real thing behaves. It follows the classic Ben Eater / SAP-1 style architecture: one shared 8-bit bus, a handful of registers sitting on that bus, and a control unit that decides who gets to talk on it at any given moment.

There's also a [Verilog port of this exact design](https://github.com/Ramakrishna-MN/8-Bit-Cpu-Verilog), built to run on a Zynq board with real switches and a real 7-segment display instead of a simulator. If you're the kind of person who wants to see this thing actually blink on hardware, that repo is where it happens. Everything explained below was cross-checked against that port, so the two describe the same machine.

<p align="center">
  <a href="https://github.com/Ramakrishna-MN/8-bit-computer-logisim/blob/main/Screenshots/CPU.png">
    <img src="https://github.com/Ramakrishna-MN/8-bit-computer-logisim/raw/main/Screenshots/CPU.png" alt="Full CPU schematic in Logisim">
  </a>
</p>

## What's actually going on here

Every functional block in this computer — the program counter, the memory address register, RAM, the instruction register, both general registers, the ALU, the output register — is wired onto **one shared bus**. Nothing has a private data path to anything else. If a register wants to hand data to another register, it goes onto the bus through a tri-state buffer, and whoever needs it reads it off the bus at the same moment. Only one thing is allowed to drive the bus at a time, and the control unit's entire job is making sure that rule never gets broken.

That single design choice is why this whole computer works off just 17 control signals instead of a tangle of dedicated wiring between every pair of components. Once you understand the bus, the rest of the schematic reads itself.

A few details of this implementation are worth calling out specifically, because they're easy to miss just by staring at the schematic, and they trip people up if they're expecting a textbook-perfect SAP-1:

**The instruction register only puts 4 bits on the bus.** It's an 8-bit register — it has to be, since it stores a full opcode-plus-operand byte when instructions are fetched from RAM — but when it drives the bus back out, only the bottom 4 bits (the operand nibble) actually go out. The top 4 bits get pulled down to zero instead. That's deliberate: by the time the operand needs to reach the bus (to address RAM, or to load an immediate value), the opcode nibble has already done its job elsewhere, so there's no reason to push it back onto the bus.

**The memory address register and the program counter are only 4 bits wide**, which makes sense given RAM only has 16 addressable locations. When either of them drives the bus, the upper 4 bits of the bus are forced to zero rather than left floating. And when either one latches a value off the bus, it only keeps the bottom 4 bits — anything in the upper nibble gets silently truncated. Worth remembering if you're ever debugging why a jump or a memory access isn't landing where you expected.

**The instruction register's upper nibble doesn't go back onto the bus at all — it goes straight into the control unit.** Those 4 bits are the opcode, and they feed directly into the low-order bits of the control ROM's address input. The remaining 3 bits of that address come from the T-state counter. In other words, the control ROM is addressed as `{opcode[3:0], tstate[2:0]}` — 7 bits, 128 words — and that's the entire mechanism deciding what the machine does on any given clock edge.

**RAM and the memory address register are clocked twice as fast as everything else.** Every other register — the PC, the IR, both general registers, the output register, the T-state counter — updates on the slow clock edge. RAM and the MAR run on a clock ticking at double that rate. The reasoning is straightforward: whatever the control unit decided last cycle needs to have already been latched into RAM or the MAR *before* the rest of the system's next edge arrives and expects that data to be valid. Running these two components ahead of the pack is what keeps the timing honest. If you're rebuilding this in an FPGA-friendly form, this can be made negative-edge triggered instead — but the values still need to be picked up from the bus on the positive edge, because that's the assumption the rest of the design is built on.

## The 17 control lines

The control unit is a ROM: 7 bits of address in (opcode + T-state), 17 bits of microcode out. Every one of those 17 output bits directly enables or disables one specific action somewhere in the machine — usually "latch the bus into this register" or "drive this register onto the bus." Here's the full map:

| Bit | Signal | What it does |
|-----|--------|---------------|
| 0 | Output register in | Latches the bus into the output register |
| 1 | ALU opcode | Selects the ALU operation — 0 = add, 1 = subtract |
| 2 | ALU output enable | Drives the ALU's result onto the bus |
| 3 | Register A in | Latches the bus into register A |
| 4 | Register A out | Drives register A onto the bus |
| 5 | Register B in | Latches the bus into register B |
| 6 | Register B out | Drives register B onto the bus |
| 7 | Halt | Stops the clock for every unit in the machine |
| 8 | Instruction register out | Drives IR[3:0] onto the bus |
| 9 | Instruction register in | Latches the full 8-bit bus into the IR |
| 10 | Memory address register out | Drives the MAR onto the bus |
| 11 | Memory address register in | Latches the bus into the MAR |
| 12 | RAM out | Drives RAM[MAR] onto the bus |
| 13 | RAM in | Latches the bus into RAM[MAR] |
| 14 | Counter in | Latches the bus into the program counter (jump) |
| 15 | Counter increment | Increments the program counter on the clock's positive edge |
| 16 | Counter out | Drives the program counter onto the bus |

Bit 16 is the most significant bit of the 17-bit control word, bit 0 is the least significant.

### Reading a control word, worked example

Every T-state of every instruction has one of these 17-bit words sitting in the control ROM, and once you know the bit order above, you can decode any of them by hand. Take `10800` (this one shows up as the very first micro-op of every instruction, since it's the start of the fetch cycle).

In binary, spread across all 17 bits, that's:

```
1 0 0 0 0 1 0 0 0 0 0 0 0 0 0 0 0
```

Reading left to right against the bit table above: bit 16 is set (**counter out** — push the program counter's value onto the bus) and bit 11 is set (**memory address register in** — latch that value into the MAR). Every other line is 0, meaning every other unit sits quietly and does nothing that cycle.

So `10800` in plain English is: *copy the program counter's address into the memory address register.* That's step one of fetching literally any instruction — before RAM can be read, the MAR needs to know which address to look at, and that address comes straight from the PC. Every instruction in this machine starts with exactly this micro-op.

## Instruction set

| Mnemonic | Opcode | Byte layout | Behavior |
|----------|--------|-------------|----------|
| `LDA n`  | `0000` | `0000 nnnn` | Load the byte at RAM address n into A |
| `ADD n`  | `0001` | `0001 nnnn` | B ← RAM[n], then A ← A + B |
| `SUB n`  | `0010` | `0010 nnnn` | B ← RAM[n], then A ← A − B |
| `STA n`  | `0100` | `0100 nnnn` | Store A into RAM address n |
| `LDI v`  | `0101` | `0101 vvvv` | Load the immediate value v directly into A |
| `JUMP n` | `1100` | `1100 nnnn` | Set the program counter to n |
| `OUT`    | `1110` | `1110 0000` | Push A's value out to the display |
| `HLT`    | `1111` | `1111 0000` | Stop the clock — the machine is done |

There's no flags register anywhere in this design, and consequently no conditional jumps. That's not an oversight — the control ROM's 7-bit address (`opcode[3:0] + tstate[2:0]`) simply leaves no room for microcode to branch on a carry or zero flag. If that's ever added, it'll mean growing the address space and adding a small flags register that captures the ALU's carry/zero output whenever the ALU drives the bus. It's on the future-improvements list below.

## Requirements

- Logisim Evolution, version 3.9.0 or newer
- A Java Runtime Environment, since Logisim Evolution runs on the JVM
- Git, if you'd rather clone than download a zip

## Getting it running

Clone the repo and open the circuit:

```
git clone https://github.com/Ramakrishna-MN/8-bit-computer-logisim.git
cd 8-bit-computer-logisim
```

Open `8_Bit_CPU.circ` directly in the Logisim Evolution application — not through a generic file browser association, but by launching Logisim Evolution itself and opening the file from inside it. Opening `.circ` files any other way tends to cause rendering issues with the custom components.

The RAM modules ship empty. There's no baked-in program — you load one in by hand, the same way you'd toggle switches on a real vintage machine, described below.

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

There's no assembler, no loader script, nothing that writes to RAM for you. You program this thing the same way you'd program an actual vintage breadboard computer — by toggling probes and pulsing the clock, one byte at a time.

**Step 1 — switch to programming mode.** Set the Program Enable probe high. This reroutes the machine so that instructions go into RAM rather than being executed.

**Step 2 — pick an address.** The 4-bit Location Probe selects where in RAM the next byte will land, anywhere from `0000` to `1111`.

**Step 3 — set the instruction.** The 8-bit Instruction Probe holds the byte you're about to write. The top 4 bits are the opcode, the bottom 4 are the operand — an address for most instructions, or an immediate value for `LDI`.

For example, to encode `LDI 5` you'd set the probe to `01010101`.

**Step 4 — write it.** Tick the clock once — one full high-then-low pulse — and that byte lands in RAM at the address you selected.

**Step 5 — repeat.** Move the Location Probe to the next address, set the next instruction, tick the clock again. Keep going until every instruction (and any raw data your program needs) is loaded.

Here's a small example program — add 5 and 10, then output the result:

| Address | Instruction | Binary   | What it does             |
|---------|-------------|----------|---------------------------|
| 0000    | LDI 5       | 01010101 | Load 5 into A |
| 0001    | ADD 15      | 00011110 | Add the value stored at address 15 |
| 0010    | OUT         | 11100000 | Push the result to the display |
| 0011    | HLT         | 11110000 | Halt |
| 1110    | (raw data)  | 00001010 | The value 10, stored ahead of time for ADD to read |

<p align="center">
  <a href="https://github.com/Ramakrishna-MN/8-bit-computer-logisim/blob/main/Screenshots/Sample%20program.png">
    <img src="https://github.com/Ramakrishna-MN/8-bit-computer-logisim/raw/main/Screenshots/Sample%20program.png" alt="Sample program loaded in hexadecimal">
  </a>
</p>

**Step 6 — run it.** Flip Program Enable back low, and the machine switches from programming mode to execution mode. Start the clock and let it run continuously (Ctrl+K in Logisim toggles this), or step through it one tick at a time if you want to watch each micro-op fire individually. Either way, keep an eye on register A and the display — you'll watch the sum build up in real time as the fetch-decode-execute cycle grinds through each instruction.

Running the program above should land you on 15.

<p align="center">
  <a href="https://github.com/Ramakrishna-MN/8-bit-computer-logisim/blob/main/Screenshots/Sample%20output.png">
    <img src="https://github.com/Ramakrishna-MN/8-bit-computer-logisim/raw/main/Screenshots/Sample%20output.png" alt="Sample output on the seven segment display">
  </a>
</p>

That's the whole loop: write assembly in your head, hand-translate it to opcodes and operands, toggle it into RAM byte by byte, and watch the control unit's microcode step through fetching, decoding, and executing it exactly the way a much older, much slower computer would have.

## Where things are headed

This is very much a living project, and there's a clear list of what's next:

- **A flags register and conditional jumps.** Right now the ALU's carry and zero outputs go nowhere — nothing in the schematic captures them. Adding a small flags register and widening the control ROM's address space (2 more bits gets you to a 9-bit / 512-word ROM) would open the door to `JC`/`JZ`-style instructions that branch the same way `JUMP` already does, just gated on a flag bit.
- **A bigger instruction set.** Only 8 of the 16 possible opcodes are used right now. `NOP`, flag-conditioned jumps, an `OUT B` to output the second register, and maybe a logical ALU op like `AND` or `OR` are all reasonable additions once there's opcode space and, in some cases, flags to support them.
- **An assembler.** Hand-translating mnemonics to binary and toggling them in one byte at a time is a great way to understand what a computer is actually doing, but it stops being fun past about ten instructions. A small tool that takes assembly text and spits out the probe sequence (or a preloadable RAM image) is on the list.
- **Keeping this repo and the Verilog port in sync.** The Verilog version was built by decoding this circuit's ROM contents directly, and a couple of small corrections came out of that process (the `LDI` control word, in particular). Those fixes are getting folded back into this `.circ` file so both repos describe the exact same machine going forward.

## About

This was built to understand, at the gate level, how a computer actually executes a program — not as a black box, but as a specific sequence of specific wires turning on and off. If you're a student, a hobbyist, or just someone who's always wondered what's really happening between "write code" and "computer does thing," this is meant to be something you can open up, poke at, and watch tick through its own logic one clock edge at a time.

## License

MIT — use it, modify it, teach with it, break it and fix it again.
