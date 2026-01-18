import tables, sequtils, strformat, times, uuids

type
  StateId = string

  Transition = object
    fromState: StateId
    toState: StateId
    condition: proc (): bool

  WorkflowEvent = object
    id: UUID
    timestamp: Time
    fromState: StateId
    toState: StateId

  Workflow = ref object
    current: StateId
    transitions: seq[Transition]
    history: seq[WorkflowEvent]

# ===== Engine =====
proc newWorkflow(start: StateId): Workflow =
  Workflow(
    current: start,
    transitions: @[],
    history: @[]
  )

proc addTransition(
  wf: Workflow,
  fromState, toState: StateId,
  condition: proc (): bool
) =
  wf.transitions.add Transition(
    fromState: fromState,
    toState: toState,
    condition: condition
  )

proc step(wf: Workflow) =
  for t in wf.transitions:
    if t.fromState == wf.current and t.condition():
      let evt = WorkflowEvent(
        id: genUUID(),
        timestamp: now(),
        fromState: t.fromState,
        toState: t.toState
      )
      wf.current = t.toState
      wf.history.add evt
      return
  raise newException(ValueError,
    fmt"No valid transition from state '{wf.current}'")

# ===== Replay =====
proc replay(wf: Workflow) =
  if wf.history.len == 0:
    return
  wf.current = wf.history[0].fromState
  for e in wf.history:
    wf.current = e.toState

# ===== Audit =====
proc audit(wf: Workflow) =
  echo "\n=== Workflow Execution Trace ==="
  for e in wf.history:
    echo fmt"{e.timestamp} | {e.fromState} -> {e.toState}"

# ===== Demo =====
when isMainModule:
  var approved = false

  let flow = newWorkflow("CREATED")

  flow.addTransition("CREATED", "VALIDATED", proc(): bool = true)
  flow.addTransition("VALIDATED", "APPROVED", proc(): bool = approved)
  flow.addTransition("APPROVED", "DEPLOYED", proc(): bool = true)

  flow.step()
  echo "Current:", flow.current

  approved = true
  flow.step()
  echo "Current:", flow.current

  flow.step()
  echo "Current:", flow.current

  audit(flow)

  echo "\nReplaying workflow..."
  flow.replay()
  echo "Final State After Replay:", flow.current
