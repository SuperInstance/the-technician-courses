# A4 — Protocol Bridges and Constrained Targets (Reference)

**Track:** Robotics Assembly  
**Source repos:** [SuperInstance/marine-gpu-edge](https://github.com/SuperInstance/marine-gpu-edge) (MEP protocol bridge), [SuperInstance/Edge-Native](https://github.com/SuperInstance/Edge-Native) (firmware VM and wire protocol)  
**Training Port phases served:** Phase 4 (Independent Technician) as reference material; optional deepening for Phase 3 (Technician)

## Why this module exists

Most installs are not one-box systems. A microcontroller reads a sensor, a Jetson runs the model, and the two must talk without assuming the cloud is available. This module is reference material for the independent technician who needs to understand — or debug — the bridge between a constrained target and a larger compute node.

## The real source material

### marine-gpu-edge: a protocol bridge in production context

[marine-gpu-edge](https://github.com/SuperInstance/marine-gpu-edge) implements a distributed GPU compute mesh for marine sensor fusion. It splits work between a workstation RTX 4050 (`eileen`) and a Jetson Orin Nano (`jetsonclaw1`). The link between them is the Marine Edge Protocol (MEP), implemented in `src/mep_bridge.cpp`.

The MEP header is 16 bytes:

```cpp
struct MEPHeader {
    uint32_t magic;
    uint16_t version;
    uint16_t type;
    uint32_t length;
    uint32_t seq;
};
```

The bridge code in `src/mep_bridge.cpp` does something every protocol bridge must do:

- `send_all()` loops until the entire buffer is transmitted or the connection closes.
- `recv_all()` loops until the requested byte count arrives or the connection closes.
- `send_msg()` writes the header, then the payload.
- `recv_msg()` validates magic and version, then reads the payload up to a bounded buffer size.

The scheduler inside the same file is the reason the bridge exists: it scores candidate nodes by thermal headroom, memory fit, load, deadline latency, precision capability, and PTX forward-compatibility. A Jetson gets FP16 work; the workstation gets FP32/TF32. This is a protocol bridge *with* intent — it is not just moving bytes, it is moving the right work to the right node.

### Edge-Native: the firmware side of the same conversation

[Edge-Native](https://github.com/SuperInstance/Edge-Native) contains a real ESP32 firmware VM and a Jetson/Python bytecode layer. The firmware side under `firmware/nexus_vm/` implements a bytecode VM in C (`vm_core.c`, `vm_opcodes.c`, `vm_validate.c`), while `firmware/wire_protocol/` implements COBS framing, CRC-16, and message dispatch.

The wire protocol files are the constrained-target complement to MEP:

- `cobs.c` / `cobs.h` — Consistent Overhead Byte Stuffing, used so that packet boundaries can be recovered on a serial link even when bytes are escaped.
- `crc16.c` / `crc16.h` — CRC-16 for integrity.
- `wire_rx.c` / `wire_tx.c` — Reception and transmission state machines.
- `msg_dispatch.c` — Routes decoded messages to handlers.

The Jetson side under `jetson/` mirrors this in Python: `wire_client/`, `agent_runtime/`, and `trust_engine/` together form the supervisory half of the same conversation.

## What to study

### 1. Why bridges fail

The two most common failure modes in protocol bridges are:

- **Partial reads/writes.** TCP and serial both present streams, not messages. `send_all` and `recv_all` exist because the OS may transfer fewer bytes than requested. A bridge that calls `send()` once and assumes the whole frame left is a bridge that will fail under load.
- **Length and version mismatches.** MEP validates `magic` and `version` before trusting `length`. Edge-Native's wire layer validates framing and CRC before dispatch. Both reject early rather than decode garbage.

### 2. The bridge as policy, not just plumbing

marine-gpu-edge's `ConstraintScheduler::score_node()` encodes deployment knowledge that a technician can read directly:

- Jetson nodes get a thermal-headroom bonus but are rejected if the task would push them above 85°C.
- FP16 tasks get a bonus on Jetson because of Tensor Core throughput.
- PTX forward-compatibility is handled explicitly: sm_87 PTX runs on sm_89, but not vice versa, so PTX kernels are routed to the workstation.

This is the bridge answering the question, "Where should this run?" not just "How do I send it?"

### 3. Reference-mode usage

A Phase 4 technician pulling this up on `jetson.local` at midnight is not reading it to learn. They are reading it because a sensor node stopped answering, and they need to know whether to suspect power, framing, CRC, message dispatch, or scheduling. Keep that use case in mind: the module is structured as a diagnostic checklist, not a tutorial.

## Comprehension questions

1. A Jetson sends a 128-byte payload to the workstation, but `recv_all()` returns 60 bytes and then 0. What does the return value of 0 mean, and what should the receiving code do next?

2. Why does MEP validate `magic` and `version` before reading the payload? What class of bug is this guarding against when one node is updated and the other is not?

3. The scheduler routes an FP32 PTX kernel to the workstation instead of the Jetson. Using the scoring logic in `ConstraintScheduler::score_node()`, explain the two independent reasons this is the correct decision.

4. Edge-Native uses COBS framing on the serial link between ESP32 and Jetson. What problem does COBS solve, and why is it especially useful on a constrained target where memory for buffering is limited?
