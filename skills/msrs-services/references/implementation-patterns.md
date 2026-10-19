# Verified implementation patterns

These snippets are abridged from proof-of-concept daemons that compiled, ran
and passed their acceptance traces. The daemons implemented the tutorial specs
and the [command gate example](examples/04-command-gate/SPEC.md). Omitted
parts are marked with comments or `/* ... */`.
Baseline: msrs `9528ecf`, cu29 `=1.0.0-rc2`, statig `0.4.1`, rustc 1.96, Linux,
2026-10-18. Re-check anything that depends on a different pin.
[msrs PR #8](https://github.com/goromal/msrs/pull/8) later fixed several of the
msrs limitations these patterns work around; each affected section ends with an
"msrs #8" note. The PoC code itself is unchanged and still valid.

Names are illustrative (`Gate`, `Cycle`, `GateOut`). Each pattern is required by
a spec decision; do not copy one whose decision your spec does not make.

## 1. Pin versions

msrs declares `cu29 = "1.0.0-rc2"`, a caret requirement. A downstream workspace
without msrs's lockfile resolves the newest compatible release (cu29 1.3.0 at
the time of testing), not rc2. Pin exactly and confirm:

```toml
[workspace.dependencies]
cu29 = "=1.0.0-rc2"
statig = "=0.4.1"
msrs-core = { git = "https://github.com/goromal/msrs", rev = "<full sha>" }
msrs-transport = { git = "https://github.com/goromal/msrs", rev = "<full sha>" }
bincode = { package = "cu-bincode", version = "2.0.2", features = ["derive", "alloc"], default-features = false }
serde = { version = "1", features = ["derive"] }
ctrlc = { version = "3.5", features = ["termination"] }   # see §7
```

```bash
cargo tree -i cu29 --depth 0   # must print exactly the pinned version
```

msrs #8 pins `=1.0.0-rc2` / `=0.4.1` in its own workspace and enables
`ctrlc/termination` from `msrs-core`, so the `ctrlc` line is only needed on
older msrs pins. Keep the service's own exact pins either way.

The PoCs also built and passed on cu29 1.3.0, where `run()` is deprecated in
favor of a typed `run_until_shutdown()`. Treat an upgrade as a new baseline:
re-run the tests, daemon checks and replay gate.

## 2. Payloads

A payload is a named struct or enum with the full derive set. Its fields may be
`String`, `Vec<T>`, `Option<T>` or nested derive-complete types. Bare `String`,
`Vec<T>` and, before msrs #8, msrs's `TransportEvent<T>` are not payloads. The
last one lacked the serialization and reflection derives; msrs #8 adds them.
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

At `9528ecf`, `Effects::take_one` kept the first emission and discarded the
rest. Since msrs #8 a second emit in an `FsmTask` step is a `CuError` (and
`run_step` panics), and `FsmBatchTask` forwards every emission as one
`Batch<Out>`. A composite output built after the whole step remains the
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

The stock `FsmTask` creates `Tick(ctx.now())` only when its input is empty.
Sustained input starves ticks: 50 queued messages over 30 iterations produced
no tick. With a real clock, the tick time cannot be recovered from the log.
In the probe, an emitted tick was 1,170,037 ns while recorded `process_time.start`
was 1,169,784 ns.

When a decision depends on time, use a service-owned source task that emits a
timed payload every iteration. The time is then recorded data at the pure seam.

```rust
static EVENTS_RX: Mutex<Option<Receiver<Event>>> = Mutex::new(None);  // installed before build()

#[derive(Reflect)]
pub struct Producer { rx: Receiver<Event> }
impl Freezable for Producer {}

impl CuSrcTask for Producer {
    type Output<'m> = CuMsg<Cycle>;
    type Resources<'r> = ();

    fn new(_c: Option<&ComponentConfig>, _r: Self::Resources<'_>) -> CuResult<Self> {
        let rx = EVENTS_RX.lock().unwrap().take()
            .ok_or_else(|| CuError::from("events rx not installed"))?;
        Ok(Self { rx })
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

`IngressTask`'s channel registry is private, so a custom source task needs its
own injection slot. Draining before sampling time ensures every batched event
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

The stock `IngressTask` dequeues one message per iteration from an unbounded
channel. If input outpaces the loop, latency grows without bound. In the probe,
20 of 50 messages remained after 30 iterations. A producer like this one gives
the spec an explicit capacity and overflow policy.

msrs #8: `msrs_transport::CycleIngressTask<T>` is a stock version of this
producer. It emits `Cycle { now_ns, events, pending }` every iteration, drains
at most `max_batch` (RON config) before reading `ctx.now()`, and leaves overflow
queued (reported as `pending`) rather than dropping it. Keep a service-owned
producer when the spec needs a drop policy or a different clock domain such as
`CLOCK_MONOTONIC`. Custom tasks can now use
`msrs_transport::registry::{install, take}` instead of a private static, and
`TransportDriver::spawn_bounded` bounds the inbound channel. msrs #8 also logs
the tick time: `IngressTask` stamps `tov` every cycle and `FsmTask` ticks at it.
`examples/watchdog` in msrs is a worked example.

## 6. Daemon `main.rs`

```rust
pub type GateFsm = msrs_core::FsmTask<GateMachine>;   // crate-root aliases for RON
pub type GateEgress = EgressTask<GateOut>;

#[copper_runtime(config = "copperconfig.ron")]
struct GateApp {}
gen_cumsgs!("copperconfig.ron");

fn serve(socket: PathBuf, log: PathBuf) {
    let handles = TransportDriver::spawn(MyTransport { socket }, RtConfig::normal());
    *EVENTS_RX.lock().unwrap() = Some(handles.from_transport);   // before build()
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
| `run()` installs cu29's ctrlc handler for SIGINT only; SIGTERM killed the process (exit 143) without stopping tasks | On msrs before #8, add `ctrlc` with `features = ["termination"]`; Cargo feature unification makes cu29's handler stop on SIGTERM (msrs #8 does this from `msrs-core`). Check that SIGTERM exits 0 |
| A handler panic is caught by `run()`, which stops tasks and returns `Err` | Treat a panic as a service crash in the failure table; supervise and restart. Do not `unwrap` on input-dependent paths |
| cu29's panic hook writes `copper-crash-<ms>-<pid>.txt` to the current directory, including caught panics | Set the daemon's working directory, include crash files in retention, and avoid panicking test probes in the repo root |
| Every iteration logs a copperlist, including idle ones: about 590 B per copperlist for a three-task DAG, about 2 GB/h at 1 kHz | Specify loop rate, log location, slab size and retention. rc2 has no setting to skip idle copperlists |
| The log is split into `<base>_N.copper` slab files with a `<base>.copper` symlink | Rotate or delete the whole family; open the base name for reading |
| Killed processes leave Unix socket files behind | Remove a stale socket at bind time |
| A second `install()` for one type panics; building without `install()` returns `CuError` | Install exactly once per application build |

## 8. Tests at the pure seam

At `9528ecf`, `run_step` returned only the first emission, so it could not
prove "exactly one output." Since msrs #8 it panics on a second emission and
`msrs_core::run_step_all` returns every one; on older pins call `step`
directly:

```rust
fn step_all<M: FsmSpec>(m: &mut M, t: &Trigger<'_, M::In>) -> Vec<M::Out> {
    let mut e = Effects::default();
    m.step(t, &mut e);
    e.take()
}
```

Useful additions to scenario tests:

- an exhaustive state × event table asserting one result and invariants per pair
- shared events exercised from every leaf (`in_state(leaf)` constructors drive real transitions)
- boundary pairs `deadline - 1 ns` and `deadline`
- a seeded randomized trace asserting invariants (a small LCG avoids a dependency)
- `RobotClock::mock()` with `.with_clock(clock)` for runtime-level tests; advance
  the mock only between iterations

## 9. Replay gate

```rust
use cu29::bincode::{config::standard, decode_from_std_read};

let ids = <CuMsgs as MatchingTasks>::get_all_task_ids();
assert_eq!(ids, ["producer", "gate", "egress"]);       // slot order is load-bearing
let UnifiedLogger::Read(read) = UnifiedLoggerBuilder::new().file_base_name(&log).build()? else { panic!() };
let mut reader = UnifiedLoggerIOReader::new(read, UnifiedLogType::CopperList);
let mut machine = GateMachine::default();               // same cold state as the live run
let mut expected_id = None;
while let Ok(cl) = decode_from_std_read::<CopperList<CuMsgs>, _, _>(&mut reader, standard()) {
    if let Some(prev) = expected_id { assert_eq!(cl.id, prev, "gap in copperlist ids"); }
    expected_id = Some(cl.id + 1);
    let input = cl.msgs.0 .0.payload().cloned();
    let recorded = cl.msgs.0 .1.payload().cloned();
    let replayed = match &input {
        Some(c) => run_step(&mut machine, &Trigger::Message(c)),
        None => run_step(&mut machine, &Trigger::Tick(0)), // valid only if ticks are no-ops
        // msrs #8: replay an empty IngressTask cycle as Tick(tov) from cl.msgs.0 .0.tov
    };
    if replayed != recorded { /* report the FIRST divergence with cl.id, input and both outputs */ }
}
```

- The decode loop ends on any decode error, so it cannot tell end-of-log from
  corruption. Check contiguous copperlist IDs and compare the count with what
  the run should have produced.
- Divergence can disappear later. A wrong initial state produced 103 divergent
  copperlists before a Disable resynchronized the PoC. Fail on the first one.
- Run a negative control: perturb initialization or one input and require the
  gate to fail.
- Derive timing evidence from the same log: gaps between recorded `now_ns`,
  FSM `metadata.process_time` (`start`/`end` convert with
  `Option::<CuTime>::from`), and deadline latencies. In the gate PoC, a 1 kHz
  command stream at 100 Hz produced a 10.4 ms maximum gap, 0.026 ms step time
  and a timeout 200.1 ms after the last applied command.
