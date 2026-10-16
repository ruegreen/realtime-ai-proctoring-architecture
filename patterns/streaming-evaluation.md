# Streaming Evaluation Pattern

Server-sent events carrying model output token by token, chosen for the
interaction it enables rather than for latency.

---

## Why Streaming, Specifically

The usual argument for streaming is perceived speed. That was not the reason
here.

A completed assessment arrives after the moment to act on it has passed. The
proctor is standing in front of a team, under time pressure, and has just typed
a summary of a spoken pitch. If that summary was ambiguous, or the model has
latched onto the wrong part of it, the useful intervention happens **while the
assessment is being written**, not after it lands.

Streaming turns the proctor from a recipient of a verdict into a participant in
the evaluation. That is an interaction decision, and it produced a different
interface than a latency decision would have.

The perception argument is real but secondary: in a timed circuit, four seconds
of blank screen reads as broken, and a volunteer who thinks the tool is broken
reloads it.

---

## Shape

```
POST /chat   →   text/event-stream

handler(req, res):
    assertNotHeld(req.evaluationId, req.stationIndex)
    openStream(res)                       # headers, no buffering
    tokensIn = tokensOut = 0

    try:
        for chunk in model.stream(system: req.system, messages: req.messages):
            tokensOut += chunk.tokenCount
            emit(res, event: "token", data: chunk.text)
        emit(res, event: "done")
    catch err:
        emit(res, event: "error", data: safeMessage(err))   # explicit, not silence
    finally:
        closeStream(res)
        meter(req.evaluationId, kind: "chat", tokensIn, tokensOut)
```

Two details that matter more than they look:

- **The error is an event, not a dropped connection.** A stream that simply
  stops leaves partial text on screen that looks like a short verdict.
- **Metering happens in `finally`.** A failed call still consumed tokens, and a
  cost ledger that only records successes understates spend precisely when
  something is going wrong.

---

## Client Handling of Partial Output

```
onToken(t):   appendToTranscript(t); markState(STREAMING)
onDone():     markState(COMPLETE); enableVerdictControls()
onError(m):   markState(FAILED);   showFailureBanner(m)
              # partial text stays visible but is explicitly marked incomplete
              # verdict controls remain DISABLED
onClose():    if state == STREAMING: onError("stream ended unexpectedly")
```

The verdict controls are gated on `COMPLETE`. A proctor cannot record a score
against a truncated assessment without first re-running it — which is the whole
point of distinguishing the states.

---

## Interruption as a First-Class Action

Because the assessment renders progressively, three interventions are available
that a batch response forecloses:

| Observation mid-stream | Proctor action |
|---|---|
| Model misread the typed summary | Stop, rewrite the summary, re-send |
| Model is assessing the wrong criterion | Add clarifying context as a new exchange |
| Team said something the summary omitted | Supply it before the verdict forms |

None of these are special features. They fall out of the fact that the loop is
conversational and the output is visible while it forms.

---

## What Was Not Built

**No websocket.** The stream is one-directional and short-lived. A websocket
would add connection lifecycle state — reconnection, backoff, replay across a
container restart — for a payload that is already a request/response with
progressive output.

**No server-side session for the conversation.** Each request carries the full
message list. The server holds no per-conversation memory, so a container
restart mid-event loses nothing and any instance can serve any request.
