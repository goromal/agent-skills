---
name: msrs-services
description: Use when designing, building, reviewing, or debugging any Rust service built on the msrs framework (copper-rs + statig successor to mscpp). Covers robot-service contracts with draw.io or Excalidraw drawings, the Store/FsmSpec/FsmTask architecture, version pinning, timed inputs, daemon lifecycle, and the replay gate. Read the linked template when specifying behavior, the tutorial for worked examples, and the implementation patterns before writing code.
---

# msrs Services Development Guide

Use this skill whenever you are working on a Rust service that uses the **msrs** framework ([goromal/msrs](https://github.com/goromal/msrs)) — designing a new service from an FSM diagram, implementing it, reviewing code for compliance, or debugging behavior.

**Ground-truth note:** compiler-verified cu29/statig signatures live in the msrs repo at `docs/superpowers/notes/copper-statig-signatures.md`. That file is updated alongside the code and **wins over this skill on any conflict**. Consult it before fighting the compiler on trait shapes, payload bounds, or RON parsing.

**Verified baseline:** the runtime claims below were checked by proof-of-concept daemons built from this skill's four example specs: msrs `9528ecf`, cu29 `=1.0.0-rc2`, statig `0.4.1`, rustc 1.96 (2026-10-18). [Implementation patterns](references/implementation-patterns.md) has the abridged verified code and measurements. msrs adapter behavior (emission limits, tick time, `CycleIngressTask`, signal handling) is covered by msrs's own tests and `examples/`. Re-verify any claim that depends on a different pin.

---

## Framework Overview

**msrs** packages deterministic, FSM-driven microservices on top of [copper-rs](https://github.com/copper-project/copper-rs) (`cu29`, a data-oriented deterministic robotics runtime) and [statig](https://github.com/mdeloof/statig) (hierarchical state machines). Every service follows the same layering:

```
┌───────────────────────────────────────────────┐
│  Transport (own driver thread, RtConfig)      │  ← middleware I/O only (gRPC/zenoh/etc.)
├────────────────── channels ───────────────────┤
│  IngressTask|Producer → FsmTask → EgressTask  │  ← copper-rs DAG, deterministic, logged
│                  │                            │
│         statig FSM (handlers)                 │  ← dispatch + transitions
│                  │                            │
│         Store (pure business logic)           │  ← unit-testable in isolation
└───────────────────────────────────────────────┘
```

**The golden rule:** business logic lives exclusively in pure `Store` methods and statig state handlers, reachable through the copper-agnostic `FsmSpec::step` seam. Everything above that seam must be testable with `msrs_core::run_step` — no copper runtime, no threads, no I/O.

**The determinism rule:** handlers must not read wall-clock time, environment, or randomness. Supply time and every other decision input explicitly and record them. Replay the same ordered triggers from the same initial state/configuration and compare outputs. Replay proves reproducibility for the exercised trace; requirements, timing, and failure tests establish whether that behavior is correct.

---

## Design First: Service Contract + Editable Drawings

Every new service starts from a `SPEC.md`, written or approved by the human before implementation. Read [the service-spec template](references/service-spec-template.md) when designing or reviewing a service. For a first walkthrough, read [the tutorial](references/spec-tutorial.md); it progresses from a two-state indicator through hierarchy and time-dependent behavior to a production-style command gate with authority, batched inputs and a watchdog.

Use **draw.io (`.drawio`) or Excalidraw (`.excalidraw`)** for authored drawings, with an adjacent SVG preview. Preserve the user's chosen editor; do not replace their drawing with Mermaid. Keep editable sources and previews under version control. These text/vector design assets belong with the service spec.

The drawing and its associated contracts together form the spec. Use stable state, transition, interface, and requirement IDs to connect drawings, tables, code, and tests. A drawing need not contain every field or timing rule: refer to its contract by ID. Neither prose nor drawing silently overrides the other; resolve disagreements before implementing the affected behavior.

Choose views by what needs explanation:

- **Context/topology:** service boundaries, peers, authority, hardware, deployment and message paths. One sentence can suffice for a trivial isolated example.
- **Behavior:** states, containment, transitions, guards and effects. Distinguish command acceptance from observed physical completion.
- **Sequence/timing:** handoffs, acknowledgements, deadlines, races and failures across components when ordering matters.

Keep small specs small: combine sections or mark them not applicable with a reason. For production services, settle authority, interface semantics, event ordering, timing, failure/restart behavior, and acceptance scenarios before coding those behaviors. Record unanswered decisions with an owner and whether they block implementation; agents must not invent missing behavioral requirements.

### Contract → code traceability

| Spec element | Implementation and verification |
|--------------|---------------------------------|
| State ID and meaning | Named leaf handler, or an explicit mapping if code names differ; state invariant assertions |
| Transition ID | Source, event, guard, target, effects and rejection behavior mapped to handler(s) and scenario tests |
| Guard | Pure query with defined conditions/units, not just an unexplained method name |
| Initial state | Explicit initial leaf and initialized store/configuration; startup effects specified |
| Composite boundary | statig superstate for genuine containment; children explicitly defer shared events |
| Entry into a composite | Explicit target leaf, e.g. `Transition(State::idle())`; the drawing's initial arrow does not generate this behavior |
| Shared boundary transition | Test every descendant leaf, including precedence, effects and forbidden outputs |
| Entry/exit/internal actions | Named effects and their order. Internal transitions return `Handled`; `Transition` to the current state exits and re-enters it |
| Unexpected events | A defined result, usually rejection with a reason. statig accepts an event deferred past the top superstate without reporting it |
| Concurrent modes or service interactions | Explicit coordination, ownership and ordering contract; do not assume a composite supplies concurrency |
| Requirement ID | Acceptance scenario(s), including negative, boundary and failure cases where relevant |

Update the contract, editable drawing and preview in the same PR as behavior changes. Keep implementation refinements traceable; do not force high-level architecture into a one-node/one-Rust-handler representation.

---

## Layer 1: Store (Pure Business Logic)

A plain struct implementing the `msrs_core::Store` marker trait. Queries take `&self`, mutations take `&mut self` and stay minimal. No ports, no transport, no clock, no I/O — ever.

```rust
use msrs_core::store::Store;

#[derive(Default)]
pub struct EchoStore {
    pub count: u64,
}

impl Store for EchoStore {}

impl EchoStore {
    /// Pure query: does not mutate; caller applies the mutation separately.
    pub fn echo(&self, msg: &str) -> String {
        format!("echo {}: {}", self.count + 1, msg)
    }
}
```

**Store rules:**
- Pure queries (`&self`) compute decisions; mutations (`&mut self`) apply them — keep the two separate
- Return result structs from queries when there are multiple outputs, such as a `Decision` that is then applied
- Every method is unit-testable with a bare `Store::default()`
- Do not duplicate the FSM state in store flags. Derive reported state from `sm.state()` so invariants hold by construction

---

## Layer 2: statig Machine + FsmSpec

The statig machine dispatches events into handlers; handlers call the store and record outputs on a context struct. An `FsmSpec` impl bridges the machine to msrs.

### statig 0.4 signatures that actually compile

```rust
use statig::blocking::InitializedStateMachine;
use statig::prelude::*;

#[derive(Default)]
pub struct EchoCtx {
    pub store: EchoStore,
    pub emitted: Option<String>,   // drained by FsmSpec::step after dispatch
}

pub struct EchoEvent(pub String);

#[derive(Default)]
pub struct EchoGate;

#[state_machine(initial = "State::running()", state(derive(Debug)))]
impl EchoGate {
    #[state]
    fn running(context: &mut EchoCtx, event: &EchoEvent) -> Outcome<State> {
        context.emitted = Some(context.store.echo(&event.0));
        context.store.count += 1;
        Handled            // or Transition(State::other())
    }
}
```

Critical details:
- Context is injected **by parameter name**: the parameter must be called `context`
- `InitializedStateMachine` is **not `Default`** (init needs the context). Initialize it **eagerly** in a manual `Default` impl for the `FsmSpec` type. `init_with_context` runs the initial leaf's entry actions and those of its superstates. Lazy initialization inside `step` would run them inside the first trigger's step and attach startup effects to whichever input arrives first.

```rust
pub struct EchoMachine {
    sm: InitializedStateMachine<EchoGate>,
    ctx: EchoCtx,
}

impl Default for EchoMachine {
    fn default() -> Self {
        let mut ctx = EchoCtx::default();
        let sm = EchoGate.uninitialized_state_machine().init_with_context(&mut ctx);
        Self { sm, ctx }
    }
}
// dispatch inside step():
// self.sm.handle_with_context(&EchoEvent(msg.0.clone()), &mut self.ctx);
```

- Prefer owned (`Copy`/`Clone`) event types; borrowed events need explicit lifetimes in superstate handlers
- `statig::prelude::*` exports `Outcome`; do not give a payload that name
- The `before_transition` hook receives `(target, source)` in statig 0.4.1, opposite to its parameter names. Use `after_transition` for transition logs

### Hierarchical modes and event precedence

Use superstates for genuine containment and shared behavior: `enabled` can contain `idle` and `active`, with a common `disable` transition. Separate mode dimensions (such as authority, mission phase and health) may coexist; specify their legal combinations and arbitration instead of forcing them into a tree. Separate tasks/services are appropriate for distinct processing or ownership boundaries, with explicit message and timing contracts.

Statig dispatches **child first**. A superstate sees an event only if the child returns `Super`; hierarchy alone supplies no priority or preemption. For shared events such as disable, specify that every descendant defers them without producing conflicting effects, and test that rule. Define priority between distinct events at the deterministic dispatch seam as well: a parent handler cannot reorder an input queue. A software e-stop event also needs a defined physical outcome, latency and fault/reset contract; naming a state `disabled` does not establish those properties.

For a composite `enabled` with initial child `idle`, explicitly implement entry as `Transition(State::idle())`. statig generates no `State::enabled()` constructor for a superstate, so targeting the composite does not compile. State whether re-entry resets to `idle` or restores history; history needs explicit implementation. Keep shared events flowing upward with `Super` through intermediate superstates; `Handled` stops propagation.

Verified 0.4.1 semantics that commonly surprise:

- `Transition(State::current())` is an **external** self-transition that runs exit and entry actions. An internal transition is `Handled`.
- Sibling transitions inside a composite run only leaf exit/entry. Leaving the composite runs the leaf exit, then the superstate exit, then target entries.
- An event returned as `Super` from the top superstate is silently treated as handled. Give every event a defined result in some handler; do not rely on statig to report unhandled input.
- If a shared event's effect depends on the leaf, handle it in that leaf and let the others return `Super`. For example, `Release` from `driving` must stop motion.
- A handler panic propagates out of `FsmTask::process`. `run()` catches it, stops all tasks and returns `Err`, so the service stops. Return a rejection for input-dependent failures instead of `unwrap`/`unreachable!`.

`FsmTask` has one input and one output payload slot, and a step may emit at most once. `FsmBatchTask` forwards every emission of a step as one `Batch<Out>`. These are adapter constraints, not restrictions on conceptual service boundaries or Copper's general capabilities. Composite inputs, output batches, or a different adapter may be required. Derive latency from the actual schedule and queueing behavior; do not assume each additional task adds one tick.

See [statig's dispatch and transition semantics](https://docs.rs/statig/0.4.1/statig/#implementation). Consult the msrs compiler-verified notes for signatures; its existing flat-machine example does not by itself verify every hierarchical pattern.

### FsmSpec: the copper-agnostic seam

```rust
use msrs_core::{Effects, FsmSpec, Trigger};

impl FsmSpec for EchoMachine {
    type In = EchoMsg;
    type Out = EchoMsg;

    fn step(&mut self, trigger: &Trigger<'_, EchoMsg>, effects: &mut Effects<EchoMsg>) {
        if let Some(msg) = trigger.message() {
            // dispatch into the statig machine, then drain ctx.emitted
            // and effects.emit(...) the result
        }
        // Trigger::Tick(nanos): fires only on empty cycles, so it is starved by
        // sustained input; carry decision time in a Cycle payload instead
        // (see the timed services pattern below)
    }
}
```

### Check adapter capabilities before promising behavior

Re-check the service's pinned msrs revision before choosing an implementation; record any capability gap in its spec rather than weakening its requirements.

- **Tick starvation:** `FsmTask::process` still chooses `Message` when input is present and `Tick` only otherwise. In a probe, 50 queued messages over 30 iterations produced zero ticks. Watchdogs and deadlines need a bounded, recorded time mechanism that still runs under sustained input: use `CycleIngressTask` (below).
- **Tick time is logged:** `IngressTask` stamps `tov` on every cycle, empty or not, and `FsmTask` builds `Tick` from that recorded input `tov`, so a replay can rebuild ticks from the log. It falls back to `ctx.now()` only if upstream leaves `tov` unset.
- **Emission limit:** a second `Effects::emit` in one `FsmTask` step makes `process()` return a `CuError`, and `run_step` panics, so a double emit fails loudly. Use `FsmBatchTask` + `BatchEgressTask` and test with `run_step_all` when a step has several effects. Specify how a batch is applied and how partial application is handled.
- **Ingress and overload:** `IngressTask` still consumes at most one message per iteration. `TransportDriver::spawn` channels are unbounded; `spawn_bounded(transport, rt, inbound_capacity)` caps transport → DAG (a full channel blocks `send`; a transport that sheds load uses `try_send`). Define queue bounds, ordering and overflow behavior, and preserve acquisition time separately from ingress time when freshness matters.
- **Restart/replay:** the adapter has a no-op `Freezable` implementation. Do not promise restored machine/store state without implementing it; define cold-start state and what replay requires.

**Recommended pattern for timed services** (verified under sustained 1 kHz input, and in msrs's `examples/watchdog`): a source task emits one composite `Cycle { now_ns, events, .. }` payload every iteration. It drains a bounded batch of pending input, then samples the clock. The FSM never sees an empty input and never ticks. Its decision time and batch are recorded payload data. Time rules run inside the step after the batch, for example as an internal `CycleEnd` event. msrs ships this as `msrs_transport::CycleIngressTask<T>` (`Cycle { now_ns, events, pending }`, cap set by RON `config: { "max_batch": N }`), usually paired with `FsmBatchTask` and `BatchEgressTask`. It keeps the queue rather than dropping; write a service-owned producer when the spec requires a drop policy and a `dropped` count. See [implementation patterns §5](references/implementation-patterns.md#5-timed-producer-source-task).

Timeouts and rate limits compare explicit monotonic timestamps supplied to the pure seam. Define clock domain, units, ordering and reset behavior. cu29's `ctx.now()` is a calibrated CPU counter that starts at zero when its clock is created. It is not `CLOCK_MONOTONIC`, and peer processes cannot stamp times in that domain. **Never** read `SystemTime::now()` / `Instant::now()` inside handlers.

---

## Layer 3: Copper Wiring

`msrs_core::FsmTask<M>` adapts any `M: FsmSpec + Default + Send + Sync + 'static` (with `M::In/Out: CuMsgPayload`) into a copper transform task. The service binary supplies the app struct and RON config.

### Pin the runtime

A caret requirement such as `cu29 = "1.0.0-rc2"` (`^1.0.0-rc2`) lets a workspace resolve the **newest** compatible cu29: 1.3.0 when tested, not the rc2 this skill and msrs were verified on. msrs pins `=1.0.0-rc2` / `=0.4.1` itself; the service should also pin `cu29 = "=1.0.0-rc2"` and `statig = "=0.4.1"` (or the versions in your pinned msrs revision's `Cargo.lock`), pin msrs by full git revision, and check with `cargo tree -i cu29 --depth 0`. Record the resolved versions in the spec. Treat an upgrade as a new baseline that must pass the tests, daemon checks and replay gate again.

### Payload rules (the #1 compile-error source)

Copper payloads must implement `TypePath`; under cu29 rc2's default (reflect-off) build only primitives do, so **bare `String`, `Vec<T>` and other std types cannot be payloads directly**. Wrap them in a named struct or enum with the full derive set. Inside that type, fields may be `String`, `Vec<T>`, `Option<T>` or nested derive-complete types. A composite `Cycle { now_ns, events: Vec<Event>, dropped }` and an output batch both work.

```rust
use bincode::{Decode, Encode};
use cu29::prelude::Reflect;
use serde::{Deserialize, Serialize};

#[derive(Default, Debug, Clone, PartialEq, Encode, Decode, Serialize, Deserialize, Reflect)]
pub struct EchoMsg(pub String);
```

`#[derive(Default)]` on an enum needs a unit `#[default]` variant; otherwise implement `Default` by hand. Generic payloads such as `Cycle<T>` and `Batch<T>` work; name them through crate-root aliases (`pub type FeedCycle = Cycle<Feed>;`, `msg: "crate::FeedCycle"`). `TransportEvent<T>` derives the payload set and is a payload whenever `T` is (see Transport Integration).

### Boilerplate that must exist

```rust
// build.rs — REQUIRED or #[copper_runtime] fails to expand
fn main() {
    println!("cargo:rustc-env=LOG_INDEX_DIR={}", std::env::var("OUT_DIR").unwrap());
}
```

```rust
// main.rs
pub type EchoIngress = IngressTask<EchoMsg>;          // crate-root aliases: RON's
pub type EchoFsm = msrs_core::FsmTask<EchoMachine>;   // type: field cannot parse
pub type EchoEgress = EgressTask<EchoMsg>;            // generic paths

#[copper_runtime(config = "copperconfig.ron")]
struct EchoApplication {}

gen_cumsgs!("copperconfig.ron");   // needed to read the unified log back
```

```ron
// copperconfig.ron — msg: must be FULLY qualified ("crate::...") because
// gen_cumsgs! emits into a nested module
(
    tasks: [
        ( id: "ingress", type: "EchoIngress" ),
        ( id: "echo",    type: "EchoFsm" ),
        ( id: "egress",  type: "EchoEgress" ),
    ],
    cnx: [
        ( src: "ingress", dst: "echo",   msg: "crate::EchoMsg" ),
        ( src: "echo",    dst: "egress", msg: "crate::EchoMsg" ),
    ],
    runtime: ( rate_target_hz: 100 ),   // loop pacing; applied only by run()
)
```

Manual stepping (tests and harnesses): `start_all_tasks()` → `run_one_iteration()` × N → `stop_all_tasks()`, then **`drop(application)`** to flush the unified log before reading it. `run_one_iteration()` does not apply `rate_target_hz`. For deterministic runtime-level tests, build with `.with_clock(clock)` from `RobotClock::mock()` and advance the mock only between iterations.

### Daemon lifecycle

A long-running service calls `app.run()`: it starts tasks, loops at `rate_target_hz`, and on shutdown stops tasks and returns. Then drop the app to flush the log and join the transport. Measured behavior on rc2:

- **Signals:** `run()` installs cu29's ctrlc handler, which by default catches SIGINT only. SIGTERM, which systemd sends, killed the process with exit 143 without stopping tasks. `msrs-core` depends on `ctrlc` with `termination`, so feature unification makes cu29's handler cover SIGTERM for every consumer. Verify that SIGTERM exits 0.
- **Panics:** a panic in any task is caught by `run()`, which stops tasks and returns `Err`. cu29's panic hook writes `copper-crash-<ms>-<pid>.txt` into the **current directory**, even for panics the program catches. Set the working directory and supervise restarts.
- **Log volume:** every iteration writes a copperlist, including idle ones. A three-task DAG used about 590 B per copperlist, about 2 GB/h at 1 kHz. rc2 has no setting to skip idle copperlists. Specify loop rate, log location, slab size and retention. Logs are slab files `<base>_0.copper`, `<base>_1.copper`, … plus a `<base>.copper` symlink; rotate or delete the whole family.
- **Latency:** with source → FSM → sink in one iteration, round trip was about one loop period: roughly 1 ms at 1 kHz over a Unix socket. Derive deadlines from the configured rate and measured step times.
- **Stale endpoints:** a killed daemon leaves its Unix socket file behind; remove it at bind time.

See [implementation patterns §6–7](references/implementation-patterns.md#6-daemon-mainrs) for the verified `main.rs`.

---

## Transport Integration

The `Transport` trait is the only place middleware lives. It runs on its own driver thread and speaks to the DAG through crossbeam channels — it never touches copper internals.

```rust
pub trait Transport: Send + 'static {
    type Inbound: Send + 'static;    // middleware → DAG
    type Outbound: Send + 'static;   // DAG → middleware
    fn run(self, rx_out: Receiver<Self::Outbound>, tx_in: Sender<Self::Inbound>)
        -> Result<(), String>;
}
```

Wiring order matters — channels are installed into a process-global registry that the copper tasks' `new()` consumes:

```rust
let handles = TransportDriver::spawn(transport, RtConfig::normal()); // or Fifo(prio) + pinning
IngressTask::<EchoMsg>::install(handles.from_transport);   // BEFORE the app is built
EgressTask::<EchoMsg>::install(handles.to_transport);
let mut app = EchoApplication::builder()...build()?;
```

- `install()` **panics on double-install** and `new()` **consumes** the slot — building a second app (e.g. for replay) requires a fresh `install()` first. Building without `install()` returns a clear `CuError`
- Shutdown is channel disconnect: drop the app, then `handles.join.join()`. A transport that polls its listener must also check for `Disconnected` on its outbound receiver, or it will never exit
- Transport faults that affect behavior must reach the DAG as **recorded inputs**, not be swallowed on the driver thread. Use `TransportEvent<T>` or a service-owned enum such as `Inbound::{Cmd(..), TransportError(String)}` with the payload derives. Wire-decoding failures can be answered by the transport without entering the DAG, but the spec must say so
- A custom source task, such as a service-owned producer, uses the same injection: `msrs_transport::registry::install::<MyTask, _>(rx)` before `build()` and `registry::take::<MyTask, _>()?` in `new()` (slots are keyed by task type)

---

## Testing & the Bring-Up Gate

### Unit: transitions, invariants and scenarios

`msrs_core::run_step` drives one FSM step with no runtime:

```rust
#[test]
fn idle_activates_on_foo_cmd() {
    let mut m = MyMachine::default();
    let cmd = FooCmd::new(...);
    assert_eq!(run_step(&mut m, &Trigger::Message(&cmd)), Some(expected_output));
    assert_eq!(m.state_name(), "Active");   // expose a Debug view of sm.state()
}

#[test]
fn tick_emits_nothing_when_idle() {
    let mut m = MyMachine::default();
    assert_eq!(run_step(&mut m, &Trigger::Tick(123)), None);
}
```

When the contract says "exactly one" or "none", assert on every emission with `msrs_core::run_step_all`, which returns them all in order. `run_step` panics on a second emission, so it suits single-output machines:

```rust
assert_eq!(run_step_all(&mut m, &Trigger::Message(&cmd)), vec![expected_output]);
```

Cover every specified transition, guard rejection, internal action, and time boundary, including `deadline - 1` and `deadline`. Test shared events from every descendant state using constructors that reach each leaf through real transitions. Also test competing-event order within one step, required and forbidden effects, and invariants across multi-step scenarios. Two cheap, high-yield additions: an exhaustive state × event table asserting one result and the invariants for every pair, and a seeded randomized trace asserting invariants. Test store queries directly. Edge coverage alone does not demonstrate lifecycle, timing, or physical behavior; add transport/simulation/hardware evidence appropriate to the contract.

### The replay-determinism gate (required before declaring bring-up done)

1. **Capture the complete decision trace:** drive acceptance scenarios through the service's real transport and record initial state/configuration and versions, every ordered trigger (including exact tick timestamps, faults, resets and empty cycles if meaningful), and outputs. Record any arbitration result that depends on external arrival order. Verify that the log actually contains the inputs needed to replay the pure seam; task payloads alone may omit time.
2. **Read back:** decode the recorded copperlists (`gen_cumsgs!` + `UnifiedLoggerBuilder` read mode + `decode_from_std_read::<CopperList<CuMsgs>, _, _>`). Under the documented cu29 rc2 configuration, do **not** depend on `cu29-export`: its reflect feature conflicts with the transport tasks.
3. **Replay:** initialize the same state/configuration and invoke `run_step` with the exact recorded trigger sequence. Never substitute replay-machine wall time. Restore a checkpoint only if complete state restoration is implemented; otherwise replay from startup.
4. **Diff:** compare the specified serialized effect payloads, ordering and absence of output per step. Require bit-identical results for that representation, and report the **first** divergence with its copperlist ID. Divergence can disappear later: a wrong initial state in the gate PoC produced 103 divergent copperlists before a Disable resynchronized it, so a final-state match proves nothing. Diagnose missing trace data, wrong initialization, implementation/version drift and nondeterminism before changing the check. Define separately any timing metadata produced outside the pure seam.
5. **Prove the gate works:** the `decode_from_std_read` loop stops at the first decode error, so end-of-log and corruption look the same. Check that copperlist IDs are contiguous and that the count matches the run. Run a negative control, such as a perturbed initial state or input, and require the gate to fail.
6. **Check requirements:** run scenario assertions and deadline measurements as well. Reproducing an incorrect trace does not make the service correct. When time is a payload field, derive timing evidence from the same log: cycle gaps from `now_ns`, step time from the FSM slot's `metadata.process_time`, and deadline latencies.

msrs's `examples/echo` replays every ingress cycle, with empty ones as `Tick` at their logged `tov`. `examples/watchdog` replays a time-dependent trip decision from recorded `Cycle` payloads under the real clock. Both are scaffolding: neither checks contiguous copperlist IDs or runs a negative control, so neither by itself completes a service's gate. Replaying an empty input as `Tick(0)` is valid only when the machine ignores ticks. See [implementation patterns §9](references/implementation-patterns.md#9-replay-gate).

---

## Anti-Patterns

- **I/O or clock reads inside Store or handlers** — breaks the replay gate. Time is an explicit recorded input; I/O lives in the Transport.
- **Bare `String`/`Vec<T>` as a copper payload** — no `TypePath` under reflect-off; use a derive-complete named type.
- **Unpinned cu29** — the caret requirement resolves to a newer release than msrs was verified on.
- **Lazy statig initialization inside `step`** — initial entry actions run inside the first trigger; initialize eagerly in `Default`.
- **`Transition` to the current state for an internal rule** — runs exit/entry actions; use `Handled`.
- **Deadlines driven by `Trigger::Tick`** — starved by sustained input; use a `Cycle` payload.
- **Several `emit` calls in one `FsmTask` step** — a runtime error; emit one composite output or use `FsmBatchTask`.
- **`unwrap`/`unreachable!`/panics on input-dependent paths** — a handler panic stops the whole service; reject the input instead.
- **Depending on `cu29-export`** — force-enables bevy_reflect and breaks the transport tasks; inline the log reader instead.
- **Generic paths in RON `type:`** — the codegen can't parse them; use crate-root type aliases.
- **Unqualified `msg:` in RON** — must be `"crate::MyMsg"`.
- **Reusing installed channels for a second app build** — `new()` consumed them; `install()` again.
- **Blocking in `FsmSpec::step`** — the DAG iteration stalls; anything slow belongs on the transport thread.
- **Logic in the binary crate** — `main.rs` is wiring only (aliases, app struct, transport spawn, run loop); if `main.rs` contains a decision, it can't be replayed or unit-tested.

---

## Compliance Checklist

**Spec**
- [ ] SPEC.md and editable draw.io/Excalidraw drawings with previews agree; IDs trace behavior to code/tests
- [ ] Authority, interfaces, event priority, effects, timing and failure/restart behavior are settled or explicitly not applicable
- [ ] Applicable views and acceptance scenarios exist; blocking design decisions and adapter gaps are resolved before dependent implementation
- [ ] Contract, drawing and preview updated in the same PR as any behavior change

**Architecture**
- [ ] All business logic in Store methods + statig handlers (reachable via `run_step`)
- [ ] Store has no I/O, no clock, no runtime dependencies
- [ ] Payloads are derive-complete named types; each `FsmTask` step emits at most one (composite) output, or the task is an `FsmBatchTask`
- [ ] statig machine initialized eagerly; every event has a defined result; internal transitions use `Handled`
- [ ] `main.rs` is wiring only
- [ ] cu29/statig/msrs pinned exactly and resolved versions recorded

**Determinism**
- [ ] No wall-clock/random/env reads inside the `FsmSpec` seam; complete ordered triggers and initial state/configuration are recorded
- [ ] Every time a decision depends on is recorded in a payload, not only in `Tick`
- [ ] Replay-determinism gate implemented and passing (bit-identical diff, first divergence reported, contiguous IDs, negative control fails)

**Testing**
- [ ] Transition and invariant tests include shared-event precedence, rejection, time boundaries and required/forbidden effects
- [ ] "Exactly one"/"no output" assertions use `run_step_all`
- [ ] Scenario, fault/restart and load/deadline checks satisfy the service contract
- [ ] Store queries unit-tested directly

**Wiring**
- [ ] `build.rs` exports `LOG_INDEX_DIR`
- [ ] `install()` before app build; fresh installs before any second build
- [ ] Transport faults that affect behavior enter the DAG as a service-owned payload variant, not swallowed
- [ ] SIGINT and SIGTERM both stop tasks and flush the log; working directory, log retention and stale-socket cleanup defined

---

## Feedback Protocol (mandatory at end of bring-up)

Bringing up a service on msrs is also a test of msrs. After the acceptance criteria pass (and before declaring the work done), run a short retrospective over what you hit and **route every lesson to its durable home**:

| What you learned | Where it goes |
|------------------|---------------|
| msrs API friction, bug, or missing capability | GitHub issue on `goromal/msrs` labeled `bringup-feedback` (small fixes: a PR directly) |
| A cu29/statig/copper fact that cost real time (signature, feature conflict, build quirk) | PR appending to `docs/superpowers/notes/copper-statig-signatures.md` in the msrs repo |
| This skill, its template or its examples were wrong, silent, or misleading | PR to `goromal/agent-skills` updating `skills/msrs-services/` (use the `editing-skills` skill) |
| Service-specific decisions | The service's own SPEC.md / docs |

Rules:
- **Nothing ends its life in a session transcript.** If it surprised you, it will surprise the next agent.
- One lesson, one home — don't duplicate content across the skill and the notes file; link instead.
- File the issues/PRs as part of the bring-up work itself, not as a suggestion to the user (report what you filed).
