# Writing a small service spec

These eight examples teach the contract a drawing needs around it. The first
three use virtual indicators and monitoring, so there is no motor-control or
physical-stop policy to decide. The fourth applies the full template to a small
production-style actuator gate. Examples 5–8 then build up, one concern at a
time, to a simple mission planning and execution service for a drone. Timing
values are teaching choices, not robot defaults.

Each spec was implemented as a proof-of-concept daemon on msrs and passed its
acceptance traces, live over a Unix socket and through the replay gate. The
"In code" notes below record what the implementations taught. The code is not
shipped with the skill; [implementation patterns](implementation-patterns.md)
contains the verified pieces.

Examples 1–4 draw in draw.io (`behavior.drawio`); examples 5–8 draw in
Excalidraw (`*.excalidraw`). Open the editable source in either editor, or read
the SVG preview beside the spec. The editor does not change the semantics:
both use the same IDs, legend conventions and contracts. Use whichever editor
the service's owner prefers, keep one editable source per drawing, and export
the preview from that source after every edit.

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

## 5. Desired versus observed state: a drone arming supervisor

Read [05 — A drone arming supervisor](examples/05-arming-supervisor/SPEC.md).

The service requests arming; the autopilot decides. The spec therefore keeps
three things apart: the operator's reply (`Accepted` means a request was sent),
the request itself (`ArmCmd`) and the observed state (telemetry). The arming
state ends either when telemetry reports armed or when an acknowledgement
timeout cancels the request. Cold start adopts whatever the autopilot reports,
so a restart in flight never disarms a flying drone.

Trace one cycle `[Telem battery 20, Arm]` and then `[Arm, Telem battery 20]`
from S1. Compare with A9: arrival order inside a cycle decides the answer, and
the spec says so. Then trace a late armed report after T7 (A6): the service
adopts it instead of fighting the autopilot.

In code: the stock `CycleIngressTask` → `FsmBatchTask` → `BatchEgressTask`
carried the whole contract with no custom task. Telemetry is recorded at its
position in the batch before the state's rule sees it. `Status` is built after
the step and emitted only when it differs from the last one sent, which makes
R7 true by construction. A fresh consumer lockfile exposed that the cu29 pin
did not hold cu29's sub-crates; msrs now pins them (see SKILL.md, Pin the
runtime).

## 6. A goal lifecycle: a drone goto executor

Read [06 — A drone goto executor](examples/06-goto-executor/SPEC.md) and its
[sequence drawing](examples/06-goto-executor/sequence.svg).

A goal is accepted or rejected immediately, then ends with exactly one terminal
event. A new valid goal preempts the active one; a rejected goal leaves it
alone. The service streams a setpoint every cycle while active, because the
autopilot holds position when the stream stops. Arrival is a computation with
a tolerance and a dwell time, not a state name. The pilot's mode switch is an
authority above the service.

Trace a drone that has dwelled for 1000 ms in S4 while its last pose is 300 ms
old (A9). Fault rules run before arrival rules, so the goal aborts with
`position_lost`. The spec makes this an ordering decision with two internal
events, `Supervise` then `Advance`, because statig dispatches child first.

In code: T9 (`no_progress`) belongs to the S3 leaf but ranks below the
superstate's fault rules on the same `Supervise` event. Child-first dispatch
lets the leaf win unless it explicitly defers when a fault holds; a mutation
test confirmed the bug is silent otherwise. Preemption deliberately re-enters
S3 with `Transition(State::approaching())`. That transition is leaf to leaf
inside `active`, so `active`'s entry action never runs; per-goal resets are
transition effects. The first replay negative control perturbed a pose drained
while idle and passed vacuously; perturb something the spec makes visible.

## 7. An algorithm contract: a drone mission planner

Read [07 — A drone mission planner](examples/07-mission-planner/SPEC.md).

Most of this service is a computation: validation stages, leg construction, a
segment test against a no-fly zone and an energy budget. The FSM is three
states. The spec gives exact formulas, an issue order and a worked example, so
two implementations must produce the same bytes. Validation reports every
issue instead of the first, because an operator fixes a mission in one pass.
The plan is versioned and replaced atomically; a lock protects a plan in use.

Work A1 by hand: five legs taking 10, 10, 10, 10.198039 and 40 s. Then predict
A2's seven issues in order before reading the table. Note why the geofence
needs only endpoint checks (a convex cylinder) but the no-fly zone needs the
segment test.

In code: the untimed service uses the stock `IngressTask` → `FsmBatchTask`;
empty steps are `Tick`s that change nothing, and the replay feeds them back as
`Tick` at the logged `tov`. A 64-leg plan encodes to 4767 B, so `Current` is
published on change only. The implementation forced one rule the first draft
lacked: an item with a non-finite number reports only `non_finite`.

## 8. Put it together: a drone mission service

Read [08 — A drone mission service](examples/08-mission-service/SPEC.md), with
its [context](examples/08-mission-service/context.svg) and
[contingency sequence](examples/08-mission-service/sequence.svg) drawings.

The planner of 07 becomes the first of two FSM stages in one DAG; an executor
that arms (05) and streams setpoints (06) flies the plan. New contracts: pause
and resume with explicit leg history, a contingency table with precedence and
escalation, operator link loss, and recovery after a restart in flight.

Trace the sequence drawing: battery low during a loiter returns home, the
operator's Resume is refused, then battery critical escalates to landing. Then
use the contingency table on one cycle whose telemetry shows manual mode,
battery 10 % and a geofence breach (A6): the first applicable row wins.

In code: the stages are `FsmTask<PlannerStage>` and `FsmTask<MissionMachine>`,
chained; both emit every cycle, so neither ticks. 07's lock is gone: a DAG is
acyclic within an iteration, so the executor snapshots the plan at `Start`
instead of locking the planner. Contingencies depend on the leaf, and statig
superstates do not know the active leaf, so each leaf passes its table column
to one shared `supervise` function. Replaying each stage from its own recorded
input localized the negative control to the executor (planner: 0 divergent
copperlists; executor: 425). Every first-run test failure was a test that
misread arrival order inside a cycle, not a service bug; the spec now states
those rules.

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
process cannot run. Examples 5–8 show each of these on a drone. Mark unknowns
as decisions to resolve instead of guessing.

Bring that draft and its editable drawing to the next design review. The first
implementation step is to pin versions and check that the chosen msrs revision
can support the contracts. Then turn the acceptance traces into tests at the
pure seam. Before you call bring-up done, run the daemon under supervisor
signals and pass the replay gate, including its negative control.
