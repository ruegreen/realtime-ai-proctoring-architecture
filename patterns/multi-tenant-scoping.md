# Multi-Tenant Scoping Pattern

Generalising a single hardcoded event into a platform where an evaluation is a
first-class tenant — done on the second instance, not speculatively on the
first.

---

## The Tenant

```
Evaluation
  id            stable identifier
  name          shown in the picker and control view
  stations[]    the rubric — see rubric-as-data.md
  heldAll       event-level hold
  heldStations  per-station holds
  active        whether it appears in the picker
```

Everything else — team results, attempt counts, transcripts, usage totals — is
owned by an evaluation and keyed by its id.

---

## Scope in the Route, Not the Handler

```
/api/evaluations                      tenant management (admin)
/api/evaluations/active               pickable list (any authenticated user)

/api/eval/:evalId/state               rubric, results, scores, attempts, holds
/api/eval/:evalId/result/:team
/api/eval/:evalId/transcript/...
/api/eval/:evalId/usage
```

This is the decision worth copying. Scoping could have been a filter applied
inside each handler — and then every new endpoint is one forgotten `where`
clause away from leaking another tenant's data.

Putting the id in the **path** means the scope is not available except through
the scoped route. A handler cannot accidentally query unscoped, because it has
no unscoped query to make.

```
# keys are composite, so a missing scope is a miss, not a cross-tenant hit
teamKey(evaluationId, team)                   -> "{evaluationId}::{team}"
transcriptKey(evaluationId, team, station)    -> "{evaluationId}::{team}::{station}"
usageKey(evaluationId)                        -> "{evaluationId}::totals"
```

Composite keys are the second line of defence: a query that somehow lost its
scope returns nothing rather than someone else's record.

---

## Selection Must Not Tax the Common Case

```
onLoad():
    active = fetchActiveEvaluations()
    if len(active) == 1:  select(active[0])        # no extra step at all
    elif len(active) > 1: showPicker(active)
    elif isAdmin:         promptToCreate()
    else:                 showMessage("check back shortly")
```

The original single-event experience is unchanged, because the common case is
exactly one active evaluation. Adding multi-tenancy that makes the ordinary path
longer is how a generalisation gets resented.

Admins can switch tenants mid-session; the control only appears once something
is selected.

---

## The Migration Is Idempotent by Construction

```
migrateAtBoot():
    if anyEvaluationRecordsExist(): return          # already migrated
    createEvaluation(FIRST_EVALUATION_ID, fromLegacyConfig())
    for doc in collectionsNeedingScope():
        if doc.evaluationId is absent:              # only untouched documents
            doc.evaluationId = FIRST_EVALUATION_ID
```

Two guards, both necessary:

1. **The existence check** short-circuits on an already-migrated environment.
2. **The per-document absence check** means that even if the first guard is
   bypassed, nothing already scoped is rewritten.

Together these make the migration safe to re-run and safe to ship to any
environment in any state — which matters, because a boot-time migration runs on
every deploy and every container replacement, not once.

---

## Why Defer the Generalisation

The alternative to multi-tenancy was a second deployment with the content
swapped. That forks the code and abandons the first event's history.

But the alternative to *deferring* was guessing, at the start, what would vary
between events. By the time the second event was real, the answer was known from
evidence: the rubric, the holds, the results, the cost — and nothing else.

The cost of deferring was one boot migration. The cost of guessing early would
have been the wrong abstraction baked into every route.

The generalisation was cheap only because the criteria were **already data** for
an unrelated reason — live editing during an event. Separating what varies from
what does not had already been done. See
[rubric-as-data.md](rubric-as-data.md).
