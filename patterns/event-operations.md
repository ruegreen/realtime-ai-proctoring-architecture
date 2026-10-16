# Event Operations Pattern

The controls a live event actually needs: independent pause levers, attempt
accounting that cannot be gamed, and state that survives a restart.

---

## Two Independent Hold Levers

```
canStart(evaluationId, stationIndex):
    ev = load(evaluationId)
    if ev.heldAll:                 return BLOCKED(reason: EVENT_HELD)
    if ev.heldStations[stationIndex]: return BLOCKED(reason: STATION_HELD)
    return ALLOWED
```

Either lever blocks. Releasing one does not imply releasing the other, and the
refusal names **which** is in effect — an operator who releases the station hold
and sees sessions still blocked needs to be told why without going hunting.

| Lever | Scope | Typical cause |
|---|---|---|
| Event hold | All stations | Briefing, schedule overrun, incident |
| Station hold | One station | Proctor stepped away, laptop died, queue reorganised |

Collapsing these into one control forces an operator to stop the whole circuit
to deal with one table. That is the failure the split exists to prevent.

---

## Attempt Accounting

The counter increments **at session start**, before the outcome is known.

```
startSession(evaluationId, team, stationIndex):
    gate = canStart(evaluationId, stationIndex)
    if gate is BLOCKED: return gate

    used = attemptsFor(evaluationId, team, stationIndex)
    max  = maxAttemptsFor(evaluationId, stationIndex)   # 0 == unlimited
    if max > 0 and used >= max:
        return BLOCKED(reason: ATTEMPT_LIMIT)

    incrementAttempt(evaluationId, team, stationIndex)   # ← here, not at verdict
    return OPEN
```

### Why not at the verdict

Recording at the point of outcome seems obviously correct and quietly creates an
opt-out. A team that senses the evaluation going badly simply abandons it and
returns later at no cost — and with limited attempts as the control on queue
length, that loophole eats the schedule.

Incrementing at start also survives a client crash. The verdict-time version did
not: a browser that died mid-session left no trace that the attempt happened.

### The accepted cost

A genuine technical failure consumes an attempt. This is absorbed by letting an
admin clear a team's record rather than by trying to distinguish abandonment
from failure automatically — a distinction the server cannot make and should not
guess at.

### Attempts are load-bearing twice

They cap retries, and they break leaderboard ties ascending. A team reaching a
score in one pass outranks a team reaching the same score in three.

---

## Restart Survival

Everything that would be painful to lose is persisted, not held in memory:

| State | Persisted | Consequence of loss |
|---|---|---|
| Team results and scores | yes | A morning of judging |
| Attempt counts | yes | Attempt limits become unenforceable |
| Hold state | yes | Stations silently reopen mid-briefing |
| Rubric content | yes | Event content reverts |
| Usage totals | yes | Cost visibility |
| In-flight conversation | **no** | One session must be restarted |

The conversation is deliberately not server-side state: each request carries its
own message list, so the service holds nothing per-session, any instance can
serve any request, and a restart costs at most one team re-running one station.

**Hold state in particular is a correctness issue, not a convenience.** A
container restart that cleared holds would reopen every station during whatever
the operator paused the event for.

---

## Admin Views: Poll, Fingerprint, Re-render

```
every POLL_INTERVAL:
    data = fetch(scopedStateUrl)
    fp   = fingerprint(data)
    if fp == lastFingerprint: return      # no DOM work at all
    lastFingerprint = fp
    render(data)
```

A websocket was rejected: it adds reconnection, backoff and replay-across-restart
state in exchange for freshness a leaderboard does not need. The fingerprint
check removes polling's only real cost — rebuilding a projected screen several
times a minute for unchanged data.

Up to one interval of staleness, and nothing lost in robustness.
