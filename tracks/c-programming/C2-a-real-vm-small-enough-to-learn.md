# C2 — A Real VM, Small Enough to Learn

**Track:** Programming  
**Source repo:** [SuperInstance/nexus-edge-runtime](https://github.com/SuperInstance/nexus-edge-runtime)  
**Training Port phases served:** Phase 2 (Assistant), Phase 3 (Technician), Phase 4 (Independent Technician)

## Why this module exists

Most explanations of virtual machines use toy examples or industrial monsters. The Nexus Edge Runtime is different: it is a real, deployable bytecode VM under 3,000 lines total, with an assembler, disassembler, validator, trust engine, wire protocol, safety system, and intent compiler. A trainee can read the whole thing and understand every part.

## The real source material

The VM lives in `src/core/vm.py` of [nexus-edge-runtime](https://github.com/SuperInstance/nexus-edge-runtime). The README gives the summary:

- 32-opcode stack-based VM.
- 8-byte instructions, little-endian.
- 32 registers: 16 general-purpose, 16 IO-mapped.
- 64KB addressable memory.
- 1024-deep stack.
- Assembler, disassembler, and bytecode validator included.

## What to study

### 1. The instruction format

Open `src/core/vm.py`. Every instruction is exactly 8 bytes:

```
[opcode:u8][arg8:u8][arg16:u16][imm32:i32]
```

The VM reads one instruction with:

```python
def _read_instr(self, pc: int) -> Tuple[int, int, int, int]:
    data = self.state.memory[pc:pc + self.INSTR_SIZE]
    opcode = data[0]
    arg8 = data[1]
    arg16 = struct.unpack_from('<H', data, 2)[0]
    imm32 = struct.unpack_from('<i', data, 4)[0]
    return opcode, arg8, arg16, imm32
```

This fixed-width format is deliberately simple: the VM never has to decode variable-length instructions, and the assembler never has to compute offsets.

### 2. The opcode layout

The opcodes are grouped by function:

- `0x00-0x07`: Stack (`NOP`, `PUSH_I8`, `PUSH_I16`, `PUSH_F32`, `POP`, `DUP`, `SWAP`, `ROT`)
- `0x08-0x10`: Arithmetic (`ADD_F`, `SUB_F`, `MUL_F`, `DIV_F`, `NEG_F`, `ABS_F`, `MIN_F`, `MAX_F`, `CLAMP_F`)
- `0x11-0x15`: Compare (`EQ_F`, `LT_F`, `GT_F`, `LTE_F`, `GTE_F`)
- `0x16-0x19`: Logic (`AND_B`, `OR_B`, `XOR_B`, `NOT_B`)
- `0x1A-0x1C`: I/O (`READ_PIN`, `WRITE_PIN`, `READ_TIMER_MS`)
- `0x1D-0x1F`: Control (`JUMP`, `JUMP_IF_FALSE`, `JUMP_IF_TRUE`)

Notice the design choice: all arithmetic is float. The VM is for control and sensor workloads, not integer bit-twiddling. Also notice there is no `HALT` opcode. The VM stops when `state.halted` is true or when `max_cycles` is reached.

### 3. The assembler and disassembler

The `Assembler` class compiles a small assembly language into bytecode. It does a two-pass assembly:

1. First pass collects labels.
2. Second pass encodes instructions and resolves label addresses.

For example:

```asm
PUSH_F32 42.0
WRITE_PIN 5
READ_PIN 6
GT_F
JUMP_IF_FALSE skip
PUSH_F32 99.9
WRITE_PIN 10
skip:
```

The `Disassembler` reverses the process, printing one instruction per line with addresses and decoded values. Together they make the VM inspectable: you can write code, see the bytes, and see what the VM does with them.

### 4. The validator

The `Validator` class performs static checks before deployment:

- Bytecode length must be a multiple of 8.
- Every opcode must be valid.
- Empty bytecode is rejected.

This is the first tier of the runtime's four-tier safety system (hardware, firmware, supervisory, application). The validator sits at Tier 4: it ensures that only well-formed bytecode reaches the VM.

### 5. Deployment targets

The README states the VM is designed for ESP32-S3 deployment and Jetson supervision. That means the same bytecode can run on a microcontroller in the field and be validated or monitored from a more powerful edge node. This is not an abstract educational VM. It is the execution engine the Field Kit can actually deploy.

## Comprehension questions

1. The instruction format is fixed at 8 bytes. What makes this choice easier for both the VM interpreter and the assembler, and what would break if the format were variable-width?

2. `PUSH_F32` packs a 32-bit float into the `imm32` field. In `src/core/vm.py`, find the exact `struct.pack`/`struct.unpack` calls used to encode and decode the float. Why is endianness explicit, and what kind of bug could arise if the assembler and VM disagreed on it?

3. The VM has no `HALT` opcode. How does a program terminate, and what role does `max_cycles` play in preventing runaway code?

4. The validator rejects bytecode whose length is not a multiple of 8. If you saw a payload of 26 bytes, what would you know about the sender, and which tier of the safety system would catch it?
