# Rubric as Data Pattern

Evaluation criteria live in editable records, not in source and not baked into a
prompt. Changes take effect on the next session with no restart and no deploy.

---

## The Station Record

```
Station
  name              display name for the station
  product           what is being pitched
  customerQuestion  what the customer in the scenario asks
  scenario          the situation the team is given
  task              what the team must do
  criteria[]        the pass criteria, one per line
  pushback          how the evaluator should challenge a weak answer
  stumble           hints available if the team stalls
  coaching          guidance released to the proctor AFTER the verdict
  duration          time budget for the station
```

Every one of these is content, and all of it changes up to the morning of an
event.

---

## Prompt Assembly Happens at Session Start

```
buildSystemPrompt(station):
    return compose(
        role:      EVALUATOR_ROLE,            # fixed, in source
        scenario:  station.scenario,
        question:  station.customerQuestion,
        task:      station.task,
        criteria:  station.criteria,
        pushback:  station.pushback,
        stumble:   station.stumble,
    )
    # station.coaching is deliberately EXCLUDED — see below
```

The fixed part — how to behave as an evaluator — is in source and versioned with
the code. The variable part is data. The boundary is the question *"would
changing this during an event be reasonable?"*

### Coaching notes are excluded from the prompt

`coaching` is the prepared guidance for the proctor and is released only after
the verdict is recorded. Including it in the system prompt would let the
evaluator leak the answer during the evaluation.

---

## Live Editing

```
onFieldChange(field, value):
    stageLocally(field, value)
    debounce(SAVE_DELAY, () => {
        persist(evaluationId, fullStationSet)
        showSavedIndicator(now())
    })
```

Debounced autosave rather than an explicit save button: an event operator
editing content twenty minutes before doors open should not be able to lose work
by navigating away, and should not have to think about saving at all.

**Changes apply to new sessions only.** A session already open keeps the prompt
it started with, so a mid-session edit cannot change the criteria a team is
being judged against halfway through.

---

## Reset to Known Good

```
resetToDefaults(evaluationId):
    confirm()
    persist(evaluationId, VETTED_DEFAULTS)    # in source, versioned
```

The vetted content ships with the code and is restorable in one action. This is
the mitigation for the obvious risk of live editing: a bad edit degrades
evaluations immediately and silently, and nothing in the system detects a poorly
written criterion.

Reset bounds the blast radius of a bad edit. It does **not** validate a new one,
and that limitation is stated rather than papered over.

---

## Why This Earned Its Complexity

| Without it | With it |
|---|---|
| A typo in a criterion needs a deploy | A typo needs an edit |
| Deploying during a live event | Deployment off the critical path entirely |
| One event per build | One engine, any evaluation |

The third row is the one that paid off unexpectedly. Because the criteria were
already data, generalising from a single hardcoded event to a multi-tenant
platform was a scoping pass rather than a rewrite — the hard part, separating
*what varies* from *what does not*, had already been done for a different
reason.

See [multi-tenant-scoping.md](multi-tenant-scoping.md).
