# Robot service specification template

Copy the template below into a service's `SPEC.md`. Combine sections for a small
service, and mark irrelevant concerns `N/A — <reason>`. Replace prompts with
decisions, measurable requirements, and assumptions. An unexplained method name
such as `can_start()` is not a guard specification.

Describe observable behavior before selecting Rust types or Copper tasks. Record
framework limitations in the implementation mapping; do not silently change the
required behavior to fit the current adapter. See [the tutorial](spec-tutorial.md)
for completed examples, from a two-state indicator to a production-style command
gate. For code that realizes these contracts on msrs, see
[implementation patterns](implementation-patterns.md).

## Drawing convention

- Author in draw.io or Excalidraw; commit the editable source and SVG preview with
  the spec. For example: `design/behavior.drawio` and `design/behavior.svg`.
- Use stable IDs: `S1` for a state, `T1` for a transition, `I1` for an interface,
  `R1` for a requirement and `A1` for an acceptance scenario. Scope IDs to a spec.
  A rule that changes data but not state, such as a lease or authority rule,
  may use another prefix (for example `L1`) if the spec declares it.
- Put IDs and short labels on drawings. Tables carry details. Label a transition
  `T1: event [guard] / effect`, abbreviating by reference when necessary.
- Include a legend: initial marker, composite boundary, internal event, and
  arrow meanings. In a context drawing an arrow means message flow; in a state
  drawing it means a transition. Do not use an unlabeled box for both a service
  and a state on the same view.
- An initial arrow into a composite identifies its default leaf. State whether
  re-entry uses that leaf or explicit history. Concurrent regions require their
  own coordination contract; a rectangle alone does not specify concurrency.
- Every behavioral arrow must resolve to a transition or explicitly identified
  rule. A summary view may omit detail only if it links to the detailed contract.
- Regenerate the preview after editing. Review the source, preview and tables
  together; conflicting versions block implementation of the disputed behavior.

---

```markdown
# <service-name> — Service Spec

Status: draft / ready for implementation
Owner: <design decision owner>
Spec revision: <revision>; runtime pin: msrs <full sha>, cu29 =<exact>, statig =<exact> (or pending)

## Purpose and boundary

What observable job does this service do? What is explicitly out of scope?
Where does it run: robot computer, controller, companion process, remote system?
List peers, hardware dependencies and required startup conditions.
Link a context drawing, or state the topology in one sentence for a trivial case.

## Authority and invariants

Who owns each actuator/command stream and each authoritative state estimate?
Who can command this service? How are manual/autonomous ownership and leases
transferred? What happens on conflicting or unauthorized commands?
Distinguish desired, accepted and observed state.

| ID | Invariant or required outcome | Applies when | Evidence |
|----|-------------------------------|--------------|----------|
| R1 | <precise condition, units and bound where applicable> | <scope> | A1 |

Define physical outcomes for stop, disable and faults rather than relying on
state names. Identify any independent controller/watchdog and what it guarantees
if this process, the transport or the computer stops executing.

## Configuration

| Name | Value | Units / range | Changed by / when |
|------|-------|---------------|-------------------|
| <deadline, limit, rate, capacity> | <value> | <units, valid range> | <compiled in; startup config; runtime command and its effect on in-flight state> |

Give every numeric constant that behavior depends on a row. Requirements and
transitions refer to these names. Changing a compiled-in value is a spec revision.

## Interfaces and data semantics

| ID | Direction / peer | Meaning and fields | Delivery contract |
|----|------------------|--------------------|-------------------|
| I1 | in / <peer> | <command, observation or fault; fields> | <ordering, freshness, acknowledgement> |
| I2 | out / <peer> | <desired command or observed status; fields> | <delivery and completion semantics> |

For relevant fields specify units, ranges, coordinate frames, valid/unknown
values, clock domain and whether time means acquisition or receipt. Define
source/session identity, sequence numbers, duplicate and out-of-order handling,
freshness limits and reconnection behavior where these affect decisions.
For commands distinguish acknowledgement, rejection, completion, cancellation,
retry and idempotency. State which messages may be dropped or coalesced.
Name transport/middleware, endpoints or topics, reliability/QoS and connection
lifecycle. State how command authority is enforced at the boundary where relevant.

## Behavior

Editable source: <link>; preview: <link>.
Name the initial leaf, initial data, and startup outputs. List orthogonal modes,
their legal combinations and coordination, if present.

| State ID / name | Meaning and invariant | Entry / exit effects |
|-----------------|-----------------------|----------------------|
| S1 / <name> | <observable meaning, permitted outputs> | <effects or none> |

| ID | Source | Event | Guard | Target | Ordered effects / data updates |
|----|--------|-------|-------|--------|--------------------------------|
| T1 | S1 | I1 | <actual predicate> | S2 | <including outputs> |

Define every guard's rejection behavior. When several conditions can fail at
once, state which rejection reason is reported (for example, "first failing
condition in this order"). Define a result for unexpected events; nothing in
statig reports an unhandled event. When a shared event has leaf-specific effects
(for example, release while driving must also stop motion), say which leaf
handles it.
Specify internal updates versus self-transitions, entry/exit order, composite
entry/history, and whether repeated commands reset timers or repeat effects.
Describe required and forbidden outputs, including steps with no output.
For processing/estimation/control services, also define the transformation or
algorithm contract: valid input domain, output meaning, accuracy/tolerances,
coordinate conversions and assumptions. An FSM alone does not specify the
computation performed while a state is active.

## Ordering and time

What makes one logical step? Which events can occur in it? What produces each
decision instant, and where is its time recorded? On msrs, `Trigger::Tick` is
starved by input and its time is not logged, so time-dependent decisions need
time carried in a recorded payload. Write the step as an ordered algorithm:

1. Pre-input rules (for example backward clock or overflow)
2. Inputs in arrival order, with any latches that last for the rest of the step
3. Time rules evaluated at the step's time, after inputs
4. Output construction from the resulting state

Resolve ties at deadline boundaries, including an input arriving in the same
step as a deadline. When several status reasons apply in one step, define their
precedence. State the scope of priorities: events already visible in a
batch, queued traffic, or an independent path. Define which shared events every
child must defer; superstates are child-first dispatch, not automatic overrides.

| Requirement | Bound / clock / measurement points | On violation |
|-------------|------------------------------------|--------------|
| <input age, processing deadline, watchdog, output rate> | <numeric contract> | <defined response> |

Specify rates, jitter tolerance, queue capacity, overflow/backpressure policy,
and maximum work per step. Include event-to-physical-effect latency where needed.
Define clock reset/backward-time handling and time service behavior under load.
Link a sequence/timing drawing for handoffs or races that need one.

## Failure, recovery and lifecycle

| Condition | Detection / owner | Required state and outputs | Recovery / reset authority |
|-----------|-------------------|----------------------------|----------------------------|
| <dependency loss, stale data, process restart, overload, etc.> | <including deadline> | <physical meaning where relevant> | <automatic, latched, or explicit reset> |

Specify startup readiness, graceful shutdown (which signals, including SIGTERM
from a supervisor), abrupt termination, persistent state, cold restart and
reconciliation with hardware/peer state. A panic in any task stops an msrs
service; list it as a crash with its owner and restart policy. List which
failures this service cannot detect or handle and who owns them.

## Implementation mapping and capability gaps

Map contract IDs to stores, machines, tasks/transports and output application.
Document the actual schedule and queueing; do not derive latency from task count.
State which decisions/data remain inside the deterministic seam. Explain how
multiple observations or effects fit the adapter, including partial output-batch
application. Check tick starvation, single-output limits and state restoration
against the pinned msrs revision. List required framework work explicitly.

## Deployment and operations

Name the loop rate (`runtime.rate_target_hz`), process supervisor and restart
policy, working directory (cu29 writes crash reports there), real-time
scheduling/affinity, socket or endpoint paths, and log location. Every runtime
iteration logs a copperlist, so give a log budget: bytes per copperlist ×
rate, slab size, rotation and retention. Pin and record runtime versions.

## Observability and replay

What state, transition cause, rejection reason, input age, queue/drop count and
deadline information must be visible? What status is authoritative?
Record initial state/configuration and versions, ordered triggers with exact
timestamps, and effects including no-output steps. Define replay start/checkpoint
semantics and the serialized output representation used for equality checks.
Prove required clock/ordering inputs are recoverable; payload-only echo replay
does not suffice for a timed service. The replay check reports the first
divergence, checks for gaps in copperlist IDs, and fails a negative control
(perturbed initial state or input). Name the timing evidence derived from the
log, such as cycle gaps, step times and deadline latencies.

## Acceptance scenarios

| ID / requirements | Given | When (ordered inputs and times) | Then (state, required/forbidden effects, deadline) | Evidence level |
|-------------------|-------|---------------------------------|--------------------------------------------------|----------------|
| A1 / R1, T1 | <initial state and data> | <trace> | <checkable result> | unit / transport / simulation / hardware |

Cover normal operation, guard rejection, shared transitions from every leaf,
duplicates/reordering, exact timeout boundaries (`deadline - 1` and `deadline`),
competing events within one step, failures and restart where applicable.
Include sustained-input timing tests, replay of fault/timer traces, a replay
negative control, and a supervisor-signal shutdown test. For machines with many
event/state pairs, add an exhaustive state × event invariant table and a seeded
randomized invariant trace. Link results when implemented; a planned test is
not a pass.

## Open decisions

| Decision / assumption | Owner | Blocks implementation? | Resolution / evidence needed |
|-----------------------|-------|------------------------|------------------------------|
| <unsettled behavior or capability> | <owner> | yes/no; affected part | <next step> |
```

## Readiness review

A spec is ready when another person can predict the externally visible outcome
of its acceptance traces without inventing policy. Check that every requirement
has evidence planned, every drawn transition has defined effects/rejection, and
no unresolved decision is hidden in a guard name or framework assumption.

Common gaps found when specs were implemented:

- Two conditions fail together and the spec does not name the reported reason.
- A fault rule ("treat as disable") leaves the rest of the step unspecified.
- A timed rule omits its time source or its order relative to inputs.
- A shared event's effect differs by leaf, but the spec assigns it to the composite.
- The spec gives no log budget, signal behavior or working directory for a daemon.

Human approval applies to the behavior to implement. If implementation exposes a
missing requirement, propose the specific contract change and resolve it with
the owner before implementing that behavior. Routine code organization does not
require another design approval.
