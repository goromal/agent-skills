# Verified implementation patterns

These snippets are abridged from proof-of-concept daemons that compiled, ran
and passed their acceptance traces. The daemons implemented the eight tutorial
specs, from the [indicator](examples/01-indicator/SPEC.md) to the
[mission service](examples/08-mission-service/SPEC.md). Omitted parts are
marked with comments or `/* ... */`.
Baseline: msrs `a1a8370` (examples 5–8; 1–4 on `9528ecf`), cu29 `=1.0.0-rc2`,
statig `0.4.1`, rustc 1.95, Linux, 2026-10-18. Snippets use the current msrs
API. Re-check anything that depends on a different pin.

Names are illustrative (`Gate`, `Cycle`, `GateOut`). Each pattern is required by
a spec decision; do not copy one whose decision your spec does not make.

## 1. Pin versions

A caret requirement (`cu29 = "1.0.0-rc2"`) resolves the newest compatible
release (cu29 1.3.0 at the time of testing), not rc2. msrs pins exactly; pin the
service the same way and confirm:

```toml
[workspace.dependencies]
cu29 = "=1.0.0-rc2"
statig = "=0.4.1"
msrs-core = { git = "https://github.com/goromal/msrs", rev = "<full sha>" }
msrs-transport = { git = "https://github.com/goromal/msrs", rev = "<full sha>" }
bincode = { package = "cu-bincode", version = "2.0.2", features = ["derive", "alloc"], default-features = false }
serde = { version = "1", features = ["derive"] }
```

```bash
cargo tree -i cu29-runtime --depth 0   # must print exactly 1.0.0-rc2
```

`cargo tree -i cu29` is not a sufficient check. cu29 `1.0.0-rc2` requires its
sub-crates (`cu29-runtime`, `cu29-derive`, …) with caret requirements, so a
fresh consumer lock once resolved cu29 rc2 with cu29-runtime 1.3.0:
`#[copper_runtime]` failed with `cannot find __private in cu29` while
`-i cu29` still printed rc2. msrs-core from `a1a8370` depends on every cu29
sub-crate, cu-bincode and cu-bincode-derive exactly, so a consumer that pins
msrs by revision inherits the baseline. On an older msrs revision, seed the
service's `Cargo.lock` from msrs's.

The PoCs also built and passed on cu29 1.3.0, where `run()` is deprecated in
favor of a typed `run_until_shutdown()`. Treat an upgrade as a new baseline:
re-run the tests, daemon checks and replay gate.

## 2. Payloads

A payload is a named struct or enum with the full derive set. Its fields may be
`String`, `Vec<T>`, `Option<T>` or nested derive-complete types. Bare `String`,
`Vec<T>` are not payloads; msrs's `TransportEvent<T>` is whenever `T` is.
Generic payloads (`Cycle<T>`, `Batch<T>`) work through crate-root aliases.

```rust
use bincode::{Decode, Encode};
use cu29::prelude::Reflect;
use serde::{Deserialize, Serialize};

#[derive(Default, Debug, Clone, PartialEq, Encode, Decode, Serialize, Deserialize, Reflect)]
pub struct Cycle {
    pub now_ns: u64,           // decision time travels in the payload (§5)
    pub events: Vec<Event>,    // arrival-ordered batch
    pub dropped: u32,          // overflow is data, not a log line
}

#[derive(Debug, Clone, Copy, PartialEq, Encode, Decode, Serialize, Deserialize, Reflect)]
pub enum Event { Enable(u32), Disable(u32), Vel(Vel) /* ... */ }

// `#[default]` works only on a unit variant; otherwise implement Default.
impl Default for Event {
    fn default() -> Self { Event::Disable(0) }
}
```

Use a service-owned variant such as `Inbound::TransportError(String)` when a
transport fault must be a recorded decision input. `statig::prelude::*` exports
`Outcome`, so avoid that name for a payload type.

## 3. statig machine with eager initialization

`init_with_context` runs the initial leaf's entry actions and those of its
superstates. Initialize in `Default`, not lazily inside `step`, so startup
effects do not run inside the first trigger.

```rust
use statig::blocking::InitializedStateMachine;
use statig::prelude::*;

pub struct GateMachine {
    sm: InitializedStateMachine<Gate>,
    ctx: Ctx,
}

impl Default for GateMachine {
    fn default() -> Self {
        let mut ctx = Ctx::default();
        let sm = Gate.uninitialized_state_machine().init_with_context(&mut ctx);
        Self { sm, ctx }
    }
}
```

`FsmTask<M>` requires `M: Send + Sync`; this struct satisfies it.

Verified dispatch semantics:

| Fact | Consequence |
|------|-------------|
| `Handled` stops propagation; `Super` defers to the parent | A child that handles a shared event hides it from the superstate |
| `Super` from the top superstate is silently accepted | Give unexpected events an explicit result; nothing reports them for you |
| `Transition(State::same())` exits and re-enters `same` | Implement internal transitions with `Handled` |
| `Transition(State::superstate())` does not compile | Target the composite's default leaf explicitly |
| Sibling transition exits only the leaf | `off → on` runs `exit off, enter on`; the shared superstate is untouched |
| Leaving a composite exits leaf then superstate | `on → disabled`: `exit on, exit enabled, enter disabled` |
| `before_transition` hook receives `(target, source)` despite its parameter names | Use `after_transition` for transition logging |
| Borrowed event types need explicit lifetimes in superstate handlers | Prefer owned `Copy` events |

A shared event whose effect differs by leaf is handled by that leaf; others
return `Super`. Example: `Release` from `driving` stops motion and transitions
to `idle`, while the top superstate handles it for other leaves.

```rust
#[state_machine(initial = "State::disabled()", state(derive(Debug, Clone, Copy, PartialEq, Eq)))]
impl Gate {
    #[state(superstate = "any", entry_action = "enter_disabled")]
    fn disabled(context: &mut Ctx, event: &GEv) -> Outcome<State> {
        match event {
            GEv::Ev(Event::Enable(_)) if context.latched => { context.reject(RejectReason::DisableLatched); Handled }
            GEv::Ev(Event::Enable(_)) => Transition(State::idle()),   // composite entry targets a leaf
            GEv::Ev(Event::Vel(_)) => { context.reject(RejectReason::Disabled); Handled }
            _ => Super,
        }
    }

    #[state(superstate = "enabled")]
    fn driving(context: &mut Ctx, event: &GEv) -> Outcome<State> {
        match event {
            GEv::CycleEnd if context.store.watchdog_expired(context.now_ns) => {
                context.store.stop_motion();
                Transition(State::idle())
            }
            _ => Super,
        }
    }

    #[superstate(superstate = "any")]
    fn enabled(context: &mut Ctx, event: &GEv) -> Outcome<State> {
        match event {
            GEv::Ev(Event::Disable(_)) => { context.latched = true; Transition(State::disabled()) }
            _ => Super,
        }
    }

    #[superstate]
    fn any(context: &mut Ctx, event: &GEv) -> Outcome<State> {
        match event {
            GEv::CycleEnd => Handled,
            // Every input reaching here has a defined result, including rejection.
            GEv::Ev(e) => { context.reject_unexpected(e); Handled }
        }
    }

    #[action]
    fn enter_disabled(context: &mut Ctx) { context.store.stop_motion(); }
}
```

Derive reported state from the machine instead of storing duplicate flags in
the store. A projection such as `fn gate_state(&State) -> GateState` makes an
invariant like "disabled implies zero output" true by construction.

## 4. One step, one composite output

A second emit in an `FsmTask` step is a `CuError` (and `run_step` panics);
`FsmBatchTask` forwards every emission as one `Batch<Out>`. A composite output built after the whole step remains the
simplest way to make the step's effect atomic:

```rust
impl FsmSpec for GateMachine {
    type In = Cycle;
    type Out = GateOut;
    fn step(&mut self, trigger: &Trigger<'_, Cycle>, effects: &mut Effects<GateOut>) {
        if let Some(c) = trigger.message() {
            effects.emit(self.cycle(c));   // exactly once
        }
        // Tick: unused; the producer (§5) supplies every decision instant.
    }
}

impl GateMachine {
    pub fn cycle(&mut self, c: &Cycle) -> GateOut {
        self.ctx.begin_cycle(c.now_ns);               // reset per-cycle latches/rejections
        /* spec-defined pre-batch rules: backward clock, overflow reason */
        for (i, e) in c.events.iter().enumerate() {   // arrival order
            self.ctx.index = i as u32;
            self.sm.handle_with_context(&GEv::Ev(*e), &mut self.ctx);
        }
        self.sm.handle_with_context(&GEv::CycleEnd, &mut self.ctx); // time rules after the batch
        self.ctx.output(gate_state(self.sm.state()))  // built from resulting state
    }
}
```

## 5. Timed producer source task

The stock `FsmTask` creates `Tick` only when its input is empty. Sustained
input starves ticks: 50 queued messages over 30 iterations produced no tick.

When a decision depends on time, use a source task that emits a timed payload
every iteration. The time is then recorded data at the pure seam. Start from
the stock `CycleIngressTask` (below); write a service-owned producer like this
one when the spec needs a drop policy or a different clock domain.

```rust
// before build(): msrs_transport::registry::install::<Producer, _>(handles.from_transport);

#[derive(Reflect)]
pub struct Producer { rx: Receiver<Event> }
impl Freezable for Producer {}

impl CuSrcTask for Producer {
    type Output<'m> = CuMsg<Cycle>;
    type Resources<'r> = ();

    fn new(_c: Option<&ComponentConfig>, _r: Self::Resources<'_>) -> CuResult<Self> {
        Ok(Self { rx: msrs_transport::registry::take::<Self, _>()? })
    }

    fn process(&mut self, ctx: &CuContext, out: &mut Self::Output<'_>) -> CuResult<()> {
        let (mut events, mut dropped) = (Vec::new(), 0u32);
        while let Ok(e) = self.rx.try_recv() {          // take the batch first
            if events.len() < BATCH_CAPACITY { events.push(e) } else { dropped += 1 }
        }
        let now_ns = monotonic_ns();                     // then sample time
        out.set_payload(Cycle { now_ns, events, dropped });
        out.tov = Tov::Time(ctx.now());
        Ok(())
    }
}
```

Draining before sampling time ensures every batched event
arrived before `now_ns`; later arrivals belong to the next cycle.

Choose the clock domain in the spec. `ctx.now()` is cu29's `RobotClock`, a
calibrated CPU counter that starts at zero when the clock is created. Peer
processes cannot stamp times in that domain. The PoCs used `CLOCK_MONOTONIC`
for producer and same-host clients:

```rust
fn monotonic_ns() -> u64 {
    let mut ts = libc::timespec { tv_sec: 0, tv_nsec: 0 };
    unsafe { libc::clock_gettime(libc::CLOCK_MONOTONIC, &mut ts) };
    ts.tv_sec as u64 * 1_000_000_000 + ts.tv_nsec as u64
}
```

The stock `IngressTask` dequeues one message per iteration. If input outpaces
the loop, latency grows: in the probe, 20 of 50 messages remained after 30
iterations. `TransportDriver::spawn_bounded` caps the inbound channel; a
producer like this one gives the spec an explicit capacity and overflow policy.

`msrs_transport::CycleIngressTask<T>` is the stock producer. It emits
`Cycle { now_ns, events, pending }` every iteration, drains at most `max_batch`
(RON config) before reading `ctx.now()`, and leaves overflow queued (reported as
`pending`) rather than dropping it. `IngressTask` also stamps `tov` every cycle
and `FsmTask` ticks at it, so tick time is in the log. `examples/watchdog` in
msrs is a worked example.

## 6. Daemon `main.rs`

```rust
pub type GateFsm = msrs_core::FsmTask<GateMachine>;   // crate-root aliases for RON
pub type GateEgress = EgressTask<GateOut>;

#[copper_runtime(config = "copperconfig.ron")]
struct GateApp {}
gen_cumsgs!("copperconfig.ron");

fn serve(socket: PathBuf, log: PathBuf) {
    let handles = TransportDriver::spawn(MyTransport { socket }, RtConfig::normal());
    msrs_transport::registry::install::<Producer, _>(handles.from_transport); // before build()
    EgressTask::<GateOut>::install(handles.to_transport);
    let mut app = GateApp::builder()
        .with_log_path(&log, Some(64 * 1024 * 1024))
        .expect("logger")
        .build()
        .expect("app");
    let result = app.run();          // start, loop at rate_target_hz, stop on SIGINT/SIGTERM
    drop(app);                       // flushes the unified log; closes egress
    let _ = handles.join.join();     // transport exits on channel disconnect
    result.expect("runtime");
}
```

```ron
(
    tasks: [
        ( id: "producer", type: "Producer" ),
        ( id: "gate",     type: "GateFsm" ),
        ( id: "egress",   type: "GateEgress" ),
    ],
    cnx: [
        ( src: "producer", dst: "gate",   msg: "crate::Cycle" ),
        ( src: "gate",     dst: "egress", msg: "crate::GateOut" ),
    ],
    runtime: ( rate_target_hz: 100 ),
)
```

Only `run()` applies `rate_target_hz`. A manual `run_one_iteration()` loop
does not pace itself. With source → FSM → sink in one iteration, round-trip
latency was about one loop period: roughly 1 ms at 1 kHz over a Unix socket.

## 7. Lifecycle facts for daemons

| Fact (measured) | Required handling |
|-----------------|-------------------|
| `run()` installs cu29's ctrlc handler for SIGINT only; SIGTERM killed the process (exit 143) without stopping tasks | `msrs-core` enables `ctrlc`'s `termination` feature, so feature unification makes cu29's handler stop on SIGTERM. Check that SIGTERM exits 0 |
| A handler panic is caught by `run()`, which stops tasks and returns `Err` | Treat a panic as a service crash in the failure table; supervise and restart. Do not `unwrap` on input-dependent paths |
| cu29's panic hook writes `copper-crash-<ms>-<pid>.txt` to the current directory, including caught panics | Set the daemon's working directory, include crash files in retention, and avoid panicking test probes in the repo root |
| Every iteration logs a copperlist, including idle ones. Size follows the payloads: about 590 B per copperlist in the gate PoC (about 2 GB/h at 1 kHz), 70 B for the arming supervisor's mostly empty cycles, plus 4767 B in a step that publishes a 64-leg plan | Measure bytes per copperlist from a real run; specify loop rate, log location, slab size and retention; publish large state on change. rc2 has no setting to skip idle copperlists |
| The log is split into `<base>_N.copper` slab files with a `<base>.copper` symlink | Rotate or delete the whole family; open the base name for reading |
| Killed processes leave Unix socket files behind | Remove a stale socket at bind time |
| `AF_UNIX` socket paths are limited to 108 bytes; a deep working directory failed with "path too long" | Bind relative to the working directory or under `/run` |
| A second `install()` for one type panics; building without `install()` returns `CuError` | Install exactly once per application build |

## 8. Tests at the pure seam

`run_step` panics on a second emission; `msrs_core::run_step_all` returns
every emission in order, so use it to prove "exactly one output":

```rust
assert_eq!(run_step_all(&mut m, &Trigger::Message(&c)), vec![expected]);
```

Useful additions to scenario tests:

- an exhaustive state × event table asserting one result and invariants per pair
- shared events exercised from every leaf (`in_state(leaf)` constructors drive real transitions)
- boundary pairs `deadline - 1 ns` and `deadline`
- a seeded randomized trace asserting invariants (a small LCG avoids a dependency)
- `RobotClock::mock()` with `.with_clock(clock)` for runtime-level tests; advance
  the mock only between iterations

## 9. Replay gate

`msrs_core::replay` (from `a1a8370`) supplies the copper-specific parts; the
service decides which slot is the input and how an empty slot replays.

```rust
use msrs_core::{read_copperlists, replay_gate, run_step, LogEnd, Recorded};

let ids = <CuMsgs as MatchingTasks>::get_all_task_ids();
assert_eq!(ids, ["producer", "gate", "egress"]);         // slot order is load-bearing
let (cls, end) = read_copperlists::<CuMsgs>(&log)?;       // CuMsgs from gen_cumsgs!
assert_eq!(end, LogEnd::Eof, "decoding stopped on an error, not at the end");
let recorded: Vec<_> = cls.iter().map(|cl| Recorded {
    id: cl.id,
    input: cl.msgs.0 .0.payload().expect("producer emits every iteration").clone(),
    output: cl.msgs.0 .1.payload().cloned(),             // None = no output this step
}).collect();
let mut machine = GateMachine::default();                // same cold state as the live run
let report = replay_gate(&recorded, |c| run_step(&mut machine, &Trigger::Message(c)));
assert!(report.passed(), "{:?}", report.first_divergence);  // also fails on id gaps
```

- `read_copperlists` reports how decoding ended. A clean end is
  `DecodeError::UnexpectedEnd`; corrupting 16 bytes of the copperlist section
  instead ended decoding early with `Utf8` or `UnexpectedVariant` errors, which
  it returns as `LogEnd::Error`. A record truncated at the very end still looks
  like a clean end, so compare the count with what the run should have produced.
- `replay_gate` compares bincode bytes, not `PartialEq`: a NaN in an `f64`
  payload is never equal to itself. It reports copperlist id gaps and the
  first divergence, and keeps replaying, because divergence can disappear
  later. A wrong initial state produced 103 divergent copperlists before a
  Disable resynchronized the gate PoC; a perturbed pose in the goto executor
  healed after 2.
- Behind `IngressTask`, an empty input slot replays as
  `Trigger::Tick(t)` with `t` from `cl.msgs.0 .0.tov` (`Tov::Time`); behind
  `CycleIngressTask` every slot has a payload.
- Run a negative control that must fail, and make the perturbation
  observable: change an input the spec makes visible in that step's output.
  Perturbing the first pose of the goto executor, drained while idle and
  overwritten 50 ms later, produced no divergence at all.
- Derive timing evidence from the same log: gaps between recorded `now_ns`,
  FSM `metadata.process_time` (`start`/`end` convert with
  `Option::<CuTime>::from`), and deadline latencies. In the gate PoC, a 1 kHz
  command stream at 100 Hz produced a 10.4 ms maximum gap, 0.026 ms step time
  and a timeout 200.1 ms after the last applied command.

## 10. Ordered internal events and leaf-dependent shared rules

statig dispatches child first, so a leaf that handles an event hides it from
its superstate. When rules of different precedence share a time step, give
each group its own internal event and dispatch them in the specified order.
The goto executor and the mission service run fault rules before progress
rules:

```rust
for e in &cycle.events { self.sm.handle_with_context(&ev(e), &mut self.ctx); }
self.sm.handle_with_context(&Ev::Supervise, &mut self.ctx); // faults, contingencies
self.sm.handle_with_context(&Ev::Advance, &mut self.ctx);   // arrival, leg progress
```

A leaf-specific rule that ranks *below* a superstate rule on the same event
must defer explicitly; otherwise the leaf wins:

```rust
#[state(superstate = "active")]
fn approaching(context: &mut Ctx, event: &Ev) -> Outcome<State> {
    match event {
        // T9 ranks below the superstate's T6–T8 faults.
        Ev::Supervise if context.store.fault(context.now_ns).is_none()
            && context.store.no_progress(context.now_ns) => { /* abort */ Transition(State::idle()) }
        _ => Super,
    }
}
```

A superstate handler does not know which leaf is active. When a shared rule's
effect depends on the leaf, as with a contingency table whose columns are
leaves, have each leaf call one shared function with its column:

```rust
fn supervise(context: &mut Ctx, col: Column) -> Outcome<State> {
    match context.store.contingency(context.now_ns, col) {   // first applicable row
        None => Handled,
        Some(Action::ToLanding(r)) => { context.store.reason = r; Transition(State::landing()) }
        /* ... */
    }
}
// in each S2 leaf:  Ev::Supervise => supervise(context, Column::Flying),
```

A transition between leaves of one composite never runs the composite's entry
action. When a rule resets data for "the composite", such as per-goal progress
tracking on preemption, make the reset an effect of the transition.

## 11. Chained FSM stages

Two FSM tasks chain like any other copper tasks. Each stage emits every cycle,
so the downstream stage never ticks:

```ron
(
    tasks: [
        ( id: "ingress",  type: "MissionIngress", config: { "max_batch": 32 } ),
        ( id: "planner",  type: "PlannerFsm" ),    // FsmTask<PlannerStage>
        ( id: "executor", type: "ExecutorFsm" ),   // FsmTask<MissionMachine>
        ( id: "egress",   type: "MissionEgress" ),
    ],
    cnx: [
        ( src: "ingress",  dst: "planner",  msg: "crate::InCycle" ),
        ( src: "planner",  dst: "executor", msg: "crate::PlannedCycle" ),
        ( src: "executor", dst: "egress",   msg: "crate::MissionOut" ),
    ],
    runtime: ( rate_target_hz: 50 ),
)
```

The first stage consumes its own commands and forwards the rest with its
outputs (`PlannedCycle { now_ns, events, planner, pending }`). Publish large
state, such as a compiled plan, only when it changes; the downstream stage
keeps its own copy.

A DAG is acyclic within an iteration: the downstream stage cannot send a lock
or acknowledgement upstream in the same step, and a feedback channel would
apply it at least one cycle late. Prefer ownership transfer: the mission
executor copies the plan at `Start`, so later uploads cannot change the flight.

Each stage's output is its own copperlist slot, so the gate replays three
ways: the first stage from the recorded cycles, the second from the first
stage's recorded output, and both chained from the recorded cycles. A
perturbed battery value diverged the executor and the chain (425 copperlists)
but not the planner (0), which localizes a regression to a stage.
