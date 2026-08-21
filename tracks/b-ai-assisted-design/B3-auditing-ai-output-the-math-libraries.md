# B3 — Auditing AI Output: The Math Libraries

**Track:** AI-Assisted Design  
**Source repos:** [SuperInstance/kintsugi-math-c](https://github.com/SuperInstance/kintsugi-math-c), [SuperInstance/spectral-mechanics](https://github.com/SuperInstance/spectral-mechanics), [SuperInstance/spectral-music-v2](https://github.com/SuperInstance/spectral-music-v2)  
**Training Port phases served:** Phase 2 (Assistant), Phase 3 (Technician)

## Why this module exists

AI-generated code can look correct without being correct. The 2026 production-hardening round found real correctness bugs in three math libraries, each detectable by a technician who knows how to look at the right signal. This module teaches those signals.

## Case 1: kintsugi-math-c — a parameter that is never read

[kintsugi-math-c](https://github.com/SuperInstance/kintsugi-math-c) implements "kintsugi mathematics" in C: fragment reassembly, crack graphs, and resilience metrics. The function `measure_resilience()` in `src/kintsugi.c` is declared as:

```c
double measure_resilience(const CrackGraph *cg, const GoldenJoint *joints,
                         size_t joint_count);
```

The function takes a `CrackGraph *cg`, but the body computes:

```c
double total_beauty = 0;
for (size_t i = 0; i < joint_count; i++) {
    total_beauty += joints[i].beauty;
}
return total_beauty / (double)joint_count;
```

`cg` is never used. The resilience score averages joint beauty and ignores the crack graph entirely.

The detection signal: compile with `-Wextra`. GCC warns about unused parameters. An AI can write plausible-looking math; a compiler flag can prove that a parameter is dead.

## Case 2: spectral-mechanics — a number that contradicts the docstring

[spectral-mechanics](https://github.com/SuperInstance/spectral-mechanics) treats graphs as spring-mass systems. The `virial_ratio()` function in `src/lib.rs` is documented to check the virial theorem:

> Should be ~1.0 at equilibrium.

The original implementation computed an instantaneous snapshot of positions and forces rather than a time average. On a harmonic oscillator initialized with positions `[1.0, -1.0]` and zero velocity, the function returned `385.86` instead of a value near `1.0`.

The detection signal: run the code and look at the number. The docstring promised ~1.0; the output was three orders of magnitude off. No advanced math was required to know something was wrong.

## Case 3: spectral-music-v2 — a tracking array that is written but never read

[spectral-music-v2](https://github.com/SuperInstance/spectral-music-v2) models music theory as spectral graph theory. Its voice-leading code assigns pitches to voices across a chord progression. The pre-fix implementation had a greedy voice-assignment path that could let multiple voices collide on the same pitch because a tracking array was written but never consulted during assignment.

The detection signal: a test that asserts uniqueness. Once a test checks that no two voices share the same pitch at the same chord, the bug becomes impossible to hide.

## The audit pattern

These three bugs share a common structure: each has a cheap, non-mathematical smell that a technician can learn:

1. **Unused parameters or variables.** A compiler warning or a quick grep for dead stores can reveal logic that ignores its own inputs.
2. **Numbers that violate documentation.** Run the function and compare the output to what the docstring claims.
3. **Write-only tracking state.** If an array is populated but never queried, the code is not enforcing the invariant you think it is.

The technician does not need to be a mathematician. The technician needs to be someone who reads the output, reads the docstring, and runs the checks.

## Comprehension questions

1. In `measure_resilience()`, the `CrackGraph *cg` parameter is unused. What compiler flag would reveal this, and why is "the function returns a reasonable-looking float" not evidence that the function is correct?

2. The `virial_ratio()` docstring says the result "should be ~1.0 at equilibrium," but the pre-fix code returned `385.86` for a simple oscillator. Why is running the function and reading the number a more reliable signal than asking the AI whether the implementation matches the theorem?

3. A voice-leading function populates an array that tracks which pitches have already been assigned, but the array is never read during the next assignment. What invariant is the code pretending to enforce, and what kind of test would catch the failure?

4. A Phase 3 technician is reviewing AI-generated sensor-fusion code that computes a moving average. The code maintains a `sum` variable and updates it on every sample, but the average is recomputed from the raw buffer each time. Name two concrete checks the technician should run before trusting the code.
