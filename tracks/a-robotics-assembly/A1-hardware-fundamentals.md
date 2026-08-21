# A1 — Hardware Fundamentals

**Track:** Robotics Assembly  
**Source repo:** [SuperInstance/edge-equipment-catalog](https://github.com/SuperInstance/edge-equipment-catalog)  
**Training Port phases served:** Phase 0 (Pre-Observer), Phase 1 (Observer), Phase 2 (Assistant); reference mode in Phase 4 (Independent Technician)

## Why this module exists

Before a trainee ever touches a wire on a boat, they need to be able to read a hardware spec the way a technician reads it: not as marketing, but as a set of hard constraints. Will this board boot the image we need? Does it have enough RAM to run the model? Does it have the right ports? Is the power draw going to pop a breaker? This module turns those questions into a repeatable, machine-readable check.

## The real source material

The [edge-equipment-catalog](https://github.com/SuperInstance/edge-equipment-catalog) repo is deliberately small. In its `production-round3-2026-07-10` branch it contains:

- `schemas/equipment.schema.json` — a JSON Schema that defines what an equipment profile must and may contain.
- `schemas/equipment/*.json` — five verified hardware profiles: Raspberry Pi 4 Model B, Raspberry Pi 5, NVIDIA Jetson Orin Nano, NVIDIA Jetson AGX Orin, and BeagleBone Black.
- `src/compatibility.ts` — a zero-dependency TypeScript compatibility checker that returns a boolean *and* a list of human-readable reasons for every failed requirement.

This is not a toy catalog. The profiles include uncertainty notes in a `_notes` field, sourced from public documentation, with uncertain values flagged rather than faked. The checker does not say "compatible" or "not compatible" as a black box; it says *why*.

## What to study

### 1. The schema as a checklist

Open `schemas/equipment.schema.json`. The required fields are `deviceId`, `manufacturer`, `modelName`, `cpu`, `memoryMb`, `storage`, and `supportedImages`. Notice what is *not* required: `gpu`, `usb`, `power`, `firmwareCompatibility`. The schema encodes the difference between "you must know this to deploy" and "you should know this if it matters for your workload."

Pay attention to the `gpu` object. It has a required `present` boolean, then optional `cudaCores`, `tensorCores`, and `aiTops`. This reflects a real SuperInstance lesson: a GPU can be present without being an AI accelerator. The Raspberry Pi 4's VideoCore VI is "present: true" but not useful for CUDA/TensorRT workloads. The Jetson Orin Nano is "present: true" with 1024 CUDA cores, 32 Tensor cores, and a rated 40 INT8 TOPS.

### 2. The checker logic

Read `src/compatibility.ts`. The `checkCompatibility(requirements, profile)` function is pure: same inputs always produce the same outputs, and it has no side effects. This matters because it means the checker can run in provisioning scripts, in CI, or on the Field Kit itself without surprising you.

Each requirement is evaluated independently. If a requirement is omitted, it is considered met. This is the opposite of a greedy validator. The function returns:

```ts
interface CompatibilityResult {
  compatible: boolean;
  reasons: string[];
}
```

Study the helper functions:

- `totalUsbPorts(profile)` sums counts across all USB port groups.
- `maxUsbVersion(profile)` returns the highest USB revision present.
- `hasAiAccelerator(profile)` requires `gpu.present` plus at least one of `cudaCores`, `tensorCores`, or `aiTops`.

### 3. Two profiles side by side

Compare `schemas/equipment/raspberry-pi-4-model-b.json` and `schemas/equipment/nvidia-jetson-orin-nano.json`:

| Attribute | Pi 4 | Jetson Orin Nano |
|-----------|------|------------------|
| `cpu.cores` | 4 | 6 |
| `cpu.architecture` | aarch64 | aarch64 |
| `memoryMb` | 4096 | 8192 |
| `gpu.present` | true | true |
| `gpu.aiTops` | absent | 40 |
| `storage.interfaces` | ["microsd", "usb"] | ["nvme-m2", "microsd"] |
| `power.maxDrawW` | 15 | 25 |

The Pi 4 and the Jetson share an architecture but diverge sharply on AI acceleration and primary storage. A workload that requires `nvme-m2` will reject the Pi 4. A workload that requires an AI accelerator will reject the Pi 4 for a different reason. The checker returns *all* reasons, not just the first.

### 4. Reading the honesty notes

Every profile has a `_notes` field. The Jetson Orin Nano note, for example, flags that `aiTops=40` is NVIDIA's stated INT8 figure, that `typicalDrawW=15` is an approximation inside the documented 7W-25W range, and that USB port counts reflect the developer-kit carrier board. This is the module's hidden lesson: hardware data decays, and honest technicians document uncertainty rather than hide it.

## Comprehension questions

1. Run `checkCompatibility` against the Raspberry Pi 4 with these requirements: `minMemoryMb: 8192`, `requireAiAccelerator: true`, `requiredStorageInterface: "nvme-m2"`. What are the returned reasons, and which reason is about a *hard* hardware limit versus a *configuration* limit?

2. The schema does not require `gpu`. Why would a compatibility check still fail if `requireGpu: true` is passed against a profile that omits the `gpu` field entirely?

3. A captain asks, "Can this Pi 4 run cameras too?" Translate that question into a `CompatibilityRequirements` object. What does the checker say, and what additional non-catalog question would you need to ask the captain before trusting the answer?

4. Look at the `_notes` field of any two profiles. Find one claim that is sourced from public documentation and one claim that the authors explicitly flag as approximate. Why does this matter when the catalog is used in a provisioning script on a 120-foot longliner?
