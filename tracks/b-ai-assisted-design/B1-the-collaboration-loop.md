# B1 — The Collaboration Loop

**Track:** AI-Assisted Design  
**Source spec:** [SuperInstance/the-technician `02-VIBE-CODING-PHYSICAL.md`](https://github.com/SuperInstance/the-technician/blob/main/02-VIBE-CODING-PHYSICAL.md)  
**Training Port phases served:** Phase 2 (Assistant), Phase 3 (Technician); cultural foundation for Phase 4 (Independent Technician) and Phase 5 (Trainer)

## Why this module exists

The Technician Paradigm does not replace the technician with an AI. It pairs them. The skill is not prompting; it is running a structured collaboration loop in which the human supplies context and judgment, and the AI supplies synthesis and documentation discipline. This module teaches the loop itself.

## The real source material

The framework is Vibe-Coding Physical Systems (VCPS), documented in `02-VIBE-CODING-PHYSICAL.md`. It has eight stages, each one a handoff between human and AI, with git as the shared memory.

## The eight-stage loop

### 1. Discovery — the technician's walkthrough

The technician is on site. They create a project repository (for example, `vessel-ss-marie-celeste/engine-monitoring-v1`). They state the objective in plain language — "Captain wants to know if the engine room floods while they're asleep, and if the main engine is running too hot" — and capture structured photos:

- Panoramic context shots.
- Specific component details.
- Access challenges and mounting constraints.

The AI begins building a spatial and inventory model from these images. The technician is the sensor; the AI is the integrator.

### 2. Understanding — AI-powered contextual Q&A

Using vision-language models, the AI identifies objects and asks clarifying questions:

- "I see two existing bilge pumps. Should the new high-water alarm be independent or integrated with pump switch #2?"
- "The photo of the engine shows a blank gauge port on the thermostat housing. Is this available for a temperature sensor?"
- "I cannot determine the wire gauge of the existing 12V supply near the pump. Can you take a close-up of the wire markings or measure its diameter?"

This stage is where the technician's domain knowledge becomes irreplaceable. The AI cannot see the gauge port; it can only ask about it.

### 3. Design — automated system architecture generation

The AI produces a bespoke design, not a template:

- System architecture diagram.
- Bill of materials with specs and suppliers.
- Wiring diagram with vessel-specific connection points.
- Network topology.

All artifacts are committed to git. The first commit is the design, tied to the discovery photos.

### 4. Vibing — iterative human refinement

The technician reviews the design and pushes back in natural language:

- "Move the high-water sensor from the center sump to the portside low spot near the stuffing box."
- "The captain wants redundant pumps."
- "Use the existing conduit run along the starboard hull; don't create a new one."

Each significant iteration is a new commit. The AI updates diagrams and BOM, and provides rationale. The technician is not accepting output; they are steering it.

### 5. Implementation — the dynamic installation guide

The AI turns the final design into a step-by-step, photo-driven guide:

- Tool list per step.
- Detailed instructions with reference photos.
- Critical checkpoints ("Before drilling: use a stud finder...").
- AI-requested photo moments for as-built documentation.

### 6. Installation — execution with validation

The technician executes the guide. At each checkpoint they take a photo. The AI validates:

- Object verification: "Confirmed: temperature sensor correctly installed on exhaust manifold."
- State verification: "Warning: the photo shows the wire pass-through lacks a waterproof gland."

Errors are caught during installation, not after.

### 7. Testing — automated health checks

The AI generates and runs diagnostic scripts. It commands sensor reads, asks the technician to simulate fault conditions, and validates readings against expected ranges. All results are committed.

### 8. Documentation — the living git-native manual

The final deliverable is the repository itself:

- Photographic history from discovery to final test.
- Every version of diagrams and BOM.
- Installation guide with validation photos.
- Test results.
- Open issues for future work.
- A predictive maintenance schedule.

## The Photo Protocol

Visual documentation is the primary data stream. The protocol mandates:

- Auto-tagging with GPS, timestamp, and step/component ID.
- Structured capture types: overview, detail, connection, label, gauge-reading.
- Change detection by comparing maintenance photos to the `main` branch baseline.
- Storage in Git LFS or a linked object store, indexed in SQLite.

The Photo Protocol is not optional documentation hygiene. It is the data format the collaboration loop consumes.

## What a trainee must internalize

The technician is the physical-world subagent. The AI is the orchestrator, scribe, and remote expert. The loop works only when:

- The technician captures enough context for the AI to ask good questions.
- The technician pushes back when the AI's design conflicts with tacit knowledge.
- Every decision is committed, so the next technician can reconstruct the reasoning.

## Comprehension questions

1. Stage 4 is called "Vibing." What is the technician actually doing in this stage, and why is each significant iteration committed to git rather than kept in chat history?

2. The AI asks, "I cannot determine the wire gauge of the existing 12V supply near the pump. Can you take a close-up of the wire markings or measure its diameter?" Which stage is this, and why can the technician answer it but the AI cannot?

3. A captain asks a Phase 3 technician to add a vibration sensor to an existing engine-monitoring install. Using the eight-stage loop, describe the minimum set of commits that should exist by the time the install is complete.

4. The Photo Protocol requires change detection by comparing maintenance photos to the `main` branch baseline. Why is git the right storage for this, and what would be lost if the photos were stored only in a cloud album?
