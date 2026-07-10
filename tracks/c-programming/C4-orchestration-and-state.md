# C4 — Orchestration and State

**Track:** Programming  
**Source repos:** [SuperInstance/fleet-conductor](https://github.com/SuperInstance/fleet-conductor), [SuperInstance/edge-relay-agent](https://github.com/SuperInstance/edge-relay-agent)  
**Training Port phases served:** Phase 2 (Assistant), Phase 3 (Technician), Phase 4 (Independent Technician)

## Why this module exists

A single agent on a single Jetson is understandable. A fleet of agents across multiple vessels is not. This module introduces the control-plane concepts a technician needs to read the orchestration layer: state machines, desired-state reconciliation, conservation guards, and honest scope documentation.

## The real source material

### fleet-conductor: the orchestration design

[fleet-conductor](https://github.com/SuperInstance/fleet-conductor) is a Rust library that provides the conductor primitive for distributed agent fleets. Its README describes a complete architecture even though the current implementation is a stub. The design is the teaching material.

The core concepts are:

- **Desired-state reconciliation.** The conductor observes the current fleet state, compares it to the desired specification, and generates corrective actions. The pattern is borrowed from Kubernetes controllers and is idempotent: running the loop twice produces the same result.

- **Agent lifecycle state machine.** Each agent moves through states:

  ```
  Pending → Starting → Healthy ↔ Degraded → Draining → Terminated
  ```

  State transitions are guarded by predicates. For example, `Healthy → Draining` requires the conservation check.

- **Conservation-aware scheduling.** Before scaling up or down, the conductor checks the conservation invariant:

  ```
  C_current = γ + η
  ```

  If a planned action would push `C` outside `[C_min, C_max]`, the action is deferred. This prevents the fleet from collapsing diversity or overloading nodes.

### edge-relay-agent: a real edge relay with honest scope gaps

[edge-relay-agent](https://github.com/SuperInstance/edge-relay-agent) is a Python edge research relay. Unlike fleet-conductor, it is fully implemented — and its README honestly documents three scope gaps:

1. The `serve` command is a heartbeat loop, not a real socket server.
2. The `url` argument to `register-cloud` is silently unpersisted.
3. "Discovery" is a manual registry, not automatic network discovery.

The source code in `relay.py` implements the core engine. Key data structures include:

- `CloudSource` and `EdgeNode` — registered participants with capabilities and constraints.
- `ResearchQuery` — a query submitted by the cloud for edge processing.
- `EdgeFinding` — a finding submitted by an edge node for cloud consumption.
- `RelayMessage` — a routed message with direction, priority, and size.
- `ResearchRelay` — the main engine that registers sources, submits queries, batches findings, and routes messages.

The CLI in `cli.py` exposes commands like `onboard`, `serve`, `register-cloud`, `register-edge`, `route`, `discover`, `bandwidth`, and `status`. The `serve` command explicitly comments: "In a real implementation, this would start a network server loop."

## What to study

### 1. Reconciliation as a control pattern

The fleet-conductor README gives the loop in pseudocode:

```
loop:
    observed = observe_current_state()
    desired  = get_desired_state()
    diff     = compute_diff(observed, desired)
    for action in diff:
        execute(action)
    sleep(reconcile_interval)
```

This pattern is robust because it is self-correcting. If an action fails, the next loop iteration sees the failure and retries. If the network partitions, the loop resumes convergence when connectivity returns.

### 2. State machines as a way to reason about failure

The agent lifecycle FSM is not just bookkeeping. It encodes what can go wrong:

- `Starting` covers initialization failures.
- `Degraded` covers health-check failures.
- `Draining` covers graceful shutdown.
- `Terminated` covers resource cleanup.

A technician debugging a fleet should map symptoms to states. "The agent is running but returning errors" maps to `Degraded`. "The agent is stuck before serving requests" maps to `Starting`.

### 3. Conservation as a safety guard

The conservation invariant `C = γ + η` is a simple idea with powerful consequences. Before removing an agent, the conductor checks whether the remaining population still has enough diversity (`η`) and enough alignment (`γ`). This is the orchestration-level version of "never pull a load-bearing sensor without a replacement plan."

### 4. Honest documentation as a skill

edge-relay-agent is the perfect companion example. It is real, tested code whose README says exactly what it does not do. Reading a README that tells the truth about scope is a skill. Writing one is the exercise.

## Comprehension questions

1. The reconciliation loop in fleet-conductor is described as idempotent. Why does idempotency matter when the loop runs over a network that can partition and reconnect?

2. An agent in the fleet is in the `Degraded` state. Name two possible causes from the state-machine description, and name the guarded transition that must succeed before the agent can be terminated.

3. The conservation check `C = γ + η` prevents a termination action when it would push `C` outside `[C_min, C_max]`. In plain language, what is the conductor protecting against, and why might spawning a replacement agent first be the right response?

4. edge-relay-agent's README lists three documented scope gaps. Why is "the `serve` command is a heartbeat loop, not a real socket server" a more useful statement than pretending the server is production-ready? What would go wrong if a trainee deployed it assuming full TCP service?
