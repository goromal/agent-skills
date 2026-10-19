# Writing a small service spec

These four examples teach the contract a drawing needs around it. The first
three use virtual indicators and monitoring, so there is no motor-control or
physical-stop policy to decide. The fourth applies the full template to a small
production-style actuator gate. Timing values are teaching choices, not robot
defaults.

Each spec was implemented as a proof-of-concept daemon on msrs and passed its
acceptance traces, live over a Unix socket and through the replay gate. The
"In code" notes below record what the implementations taught. The code is not
shipped with the skill; [implementation patterns](implementation-patterns.md)
contains the verified pieces.

Open each `behavior.drawio` in draw.io to edit it, or read its SVG preview beside
the spec. You can author your own version in Excalidraw using the same IDs and
contracts; the editor does not change the semantics. The examples use draw.io so
there is only one editable source per drawing to maintain.

## 1. Specify what a command means

Read [01 — A virtual indicator](examples/01-indicator/SPEC.md).

It has two states and one input. The useful design decisions are small: `Set`
expresses a desired value, repeated commands return status without re-entering
the state, and restart begins off. The output reports logical state; it does not
claim that a physical lamp changed.

Trace `Set(on), Set(on), Set(off)` through the drawing and table. You should be
able to predict all three outputs. Then compare with A2. This is the minimum
useful exercise: state names plus arrows would not settle duplicate handling.

In code: T3 is `Handled`. In statig, `Transition(State::on())` from `on` is an
external self-transition that would run exit and entry actions.

## 2. Add a shared mode

Read [02 — An enabled indicator](examples/02-enabled-indicator/SPEC.md).

`enabled` contains `off` and `on`. `Disable` applies to both children. The spec
also answers questions a composite box leaves open: enabling starts off,
re-enabling preserves the current child, and a `Set` while disabled is rejected.

Trace `Enable, Set(on), Disable, Set(on), Enable`. Compare with A1–A4.
Notice that disabling an indicator is a logical operation here. A motor service
would additionally need to define its physical stopping behavior and authority.

The child-first rule matters: both enabled children must defer `Disable` without
emitting their own status first. T4 is implemented once but exercised from both
leaves. This is where hierarchy removes duplication without defining priority
between separate queued commands.

In code: T1 targets `State::off()`; a transition to the superstate does not
compile. The reported `enabled`/`on` flags come from the statig state rather
than a second copy in the store, so R1 holds by construction. A state × command
table test covers all 12 pairs.

## 3. Add time and stale data

Read [03 — A telemetry freshness monitor](examples/03-freshness-monitor/SPEC.md).

This example consumes an explicit timed cycle containing an optional sample.
That makes the contract's ordering visible and avoids relying on idle-only ticks.
The spec separates sample acquisition time from cycle time and says exactly what
happens at the freshness boundary.

Trace a sample acquired at 0 ms, then cycles at 90 and 100 ms without a sample.
Now replace the last cycle with a sample acquired at 100 ms. Compare with A1–A3:
the new observation is applied before the cycle's freshness decision.

This also introduces a real implementation gap: the stock adapter does not
produce the promised periodic cycle input. The spec identifies the work required
without pretending that a timer guard fixes scheduling. Its output is monitor
status; consumers still need their own policy when the monitor disappears.

In code: the producer is a service-owned copper source task. Each iteration it
coalesces pending samples, samples the clock, and emits a `Cycle`. No msrs
change was needed. With a 1 kHz sensor stream and a 100 Hz loop, every cycle
ran: the maximum gap was 11.9 ms and stale was reported 8.5 ms after expiry.
Because time is in the payload, replay reproduced every status.

## 4. Put it together: an actuator gate

Read [04 — A teleoperation command gate](examples/04-command-gate/SPEC.md).

This example adds what the indicators lacked: a lease that decides whose
commands count, a motor output whose zero value is the safe state, a watchdog,
and several commands arriving in one cycle. Its sections follow the full
template, including configuration and deployment.

Trace one cycle containing `Vel, Disable, Enable, Vel` from `driving` and
compare with A3. Arrival order alone does not answer it; the latch rule does.
Then read the open decisions. Writing the spec exposed that dropping the newest
events on overflow could drop a Disable. A drawing would not show that problem.

In code: a lease release must stop motion only from `driving`, so that leaf
handles it and other leaves defer to a top superstate. The watchdog is an
internal `CycleEnd` event dispatched after the batch. Rejection reasons follow
a fixed check order, so several simultaneous failures still report one
predictable reason.

## Build your own next

Copy [the template](service-spec-template.md), then start with:

1. One sentence defining responsibility and authority.
2. A context drawing showing peers and message direction.
3. A behavior drawing with initial state and stable state/transition IDs.
4. Transition/interface tables that settle effects, rejection and data meaning.
5. Three concrete traces: normal operation, one conflict or timeout, and restart.

Then fill the remaining applicable contracts. For a more complex robot service,
pay particular attention to ownership handoff, desired versus observed state,
coordinate frames, stale inputs, bounded response time, and who acts when this
process cannot run. Mark unknowns as decisions to resolve instead of guessing.

Bring that draft and its editable drawing to the next design review. The first
implementation step is to pin versions and check that the chosen msrs revision
can support the contracts. Then turn the acceptance traces into tests at the
pure seam. Before you call bring-up done, run the daemon under supervisor
signals and pass the replay gate, including its negative control.
