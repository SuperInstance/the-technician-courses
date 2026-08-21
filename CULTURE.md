# CULTURE.md — The Honesty-Marker Convention

This repository teaches the Technician Paradigm through SuperInstance's own engineering culture. The most important convention in that culture is also the simplest: we mark every claim with one of three symbols so readers know exactly what they are looking at.

## The markers

| Marker | Meaning | Use it when |
|--------|---------|-------------|
| ✅ | Exists and was independently verified | You rebuilt it, retested it, or fact-checked it yourself. Not the agent. Not the README. You. |
| ⚠️ | Exists, is real, but has known and documented gaps | The artifact works or is true as far as it goes, but it is incomplete, approximate, or scoped down. The gaps are named. |
| 🔮 | Proposal or aspiration | It does not exist yet. Do not cite it as if it does. |

These markers apply to your *own* claims first. They are not a weapon for criticizing other people's work. They are a discipline for not fooling yourself.

## Why this matters

The Technician Paradigm runs on trust propagation. A captain does not trust a box because a manual says it works. A captain trusts a technician who can say, "This reading is simulated until I hook the real sensor — here's the marker that says so." Honesty is not a confession of weakness. It is the thing that makes trust possible.

In a network where AI agents generate code, documentation, and designs, the honesty-marker convention is the fence that keeps simulated from being presented as real. It is the reason a trainee can open a repo and know whether a feature is verified, scoped-down, or imagined.

## Worked example: the gravity-well-protocol rewrite

The best illustration of the convention is [SuperInstance/gravity-well-protocol PR #1](https://github.com/SuperInstance/gravity-well-protocol/pull/1). The original README made three claims that were not true:

1. "You can observe the public test fleet immediately."
2. A **Live Demo** link to `the-fleet.casey-digennaro.workers.dev`.
3. "The entire protocol is one ~32 KB JavaScript file."

The repository contained only a README and a license. No JavaScript file existed. The live demo was a generic fleet showcase, not a demonstration of the protocol. The rewrite did not delete the interesting ideas. It labeled them accurately.

### Before

```markdown
A coordination layer for the Cocapn Fleet where agents broadcast state updates
only within a defined local region...

You can observe the public test fleet immediately. No signups or API keys are required.

**Live Demo:** [the-fleet.casey-digennaro.workers.dev]

## Quick Start

1. Fork this repository.
2. Deploy it with one click to Cloudflare Workers. The default configuration works immediately.
3. Connect any agent...

**Core Operations:**
* **Zero Dependencies:** The entire protocol is one ~32 KB JavaScript file.
```

### After

```markdown
⚠️ This is a design/concept document — no working software exists yet.
   The repository contains only this README and a license.
   Implementation is planned for the future.

✅ **Concept** — A coordination layer for the Cocapn Fleet where agents
   broadcast state updates only within a defined local region...

🔮 The broader Cocapn fleet can be seen at [the-fleet.casey-digennaro.workers.dev].
   Note: That site showcases the fleet but does **not** demonstrate this protocol.

## Quick Start (not yet implementable)

🔮 The following steps describe the intended workflow once implementation
   is complete. No deployment is possible today.

* 🔮 **Zero Dependencies:** The intended implementation is a single
    ~32 KB JavaScript file (not yet written).
```

The rewrite keeps the real design ideas — region-limited broadcasts, traffic-informed gossip, implicit liveness detection — but marks them as concept (✅) rather than deployed software. The deployment steps and the 32KB file are marked as aspiration (🔮). The whole repo is marked as a design document with gaps (⚠️).

This is the convention in action: **honesty is not deletion; it is accurate labeling.**

## How to apply the convention in this curriculum

When you write a module, a note, or a field report:

- Mark a build command as ✅ only after you have run it on a clean checkout and seen it succeed.
- Mark a hardware spec as ⚠️ if it is sourced from documentation but you have not tested it on a boat.
- Mark a roadmap item as 🔮 if it is not yet implemented, no matter how confident you are that it will be.
- If a claim later gets verified, change the marker. The marker is a statement of evidence, not a permanent grade.

## What the markers are not

- They are not emoji decoration. Do not use them for tone.
- They are not a substitute for detail. A ⚠️ claim must name its gap. A 🔮 claim must say what is missing.
- They are not optional in SuperInstance repos. If you make a claim without a marker, readers will assume you are hiding something.

## Comprehension questions

1. The original gravity-well-protocol README said the protocol could be deployed with one click and linked to a live demo. The rewrite added an ⚠️ banner and changed the demo link to 🔮. Why is the 🔮 marker more honest than simply removing the link?

2. A trainee writes a module that includes the sentence, "The Field Kit boots from NVMe in 45 seconds." What evidence would justify a ✅ marker? Under what conditions would ⚠️ or 🔮 be more appropriate?

3. A module author marks a feature as ✅ but the source repo's own README marks the same feature as ⚠️. Which marker should win in the module, and why?

4. Why is accurate labeling more valuable than deletion when a design idea is good but unimplemented? Use the gravity-well-protocol rewrite as your example.
