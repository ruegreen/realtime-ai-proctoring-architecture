# Architecture Diagrams

Detailed diagrams for the real-time AI proctoring platform. All diagrams
describe design and control flow; none describe implementation internals.

---

## 1. High-Level System Architecture

```mermaid
graph TB
    subgraph Clients["Browser — single HTML file, no build step"]
        SESSION["Session Tab<br/>(all users)"]
        TRACKER["Tracker<br/>(admin)"]
        BOARD["Leaderboard<br/>(admin)"]
        CONTROL["Station Control<br/>(admin)"]
        CONFIG["Rubric Config<br/>(admin)"]
    end

    subgraph Server["Single Node Service"]
        AUTHMW["Auth Middleware"]
        ROLE["Role Guard"]
        STREAM["SSE Evaluation Stream"]
        STATE["Evaluation-Scoped State API"]
        HOLD["Hold Controller"]
        USAGE["Cost Ledger"]
        PDF["Scorecard Generator"]
    end

    subgraph External["External"]
        IDP["Existing Identity Gateway"]
        LLM["LLM Streaming API"]
    end

    subgraph Store["Document Store"]
        EV[("Evaluations")]
        PODS[("Team Results")]
        TR[("Transcripts")]
        US[("Usage Totals")]
    end

    SESSION --> AUTHMW --> ROLE
    TRACKER & BOARD & CONTROL & CONFIG --> ROLE
    AUTHMW -.->|delegated login| IDP
    ROLE --> STREAM --> LLM
    ROLE --> STATE & HOLD & USAGE & PDF
    STREAM -->|token counts| USAGE
    PDF --> LLM
    STATE <--> EV & PODS
    STREAM -->|free capture at verdict| TR
    PDF <--> TR
    USAGE <--> US
```

---

## 2. Evaluation Session Lifecycle

The human is in the loop at two points: composing what the model sees, and
deciding the outcome. The model never writes a score.

```mermaid
sequenceDiagram
    participant P as Proctor
    participant C as Client
    participant S as Server
    participant M as Model
    participant D as Store

    P->>C: Select team + station, Start
    C->>S: Check hold state
    alt station or event held
        S-->>C: Blocked, with the specific reason
    end
    S->>D: Increment attempt (BEFORE outcome is known)
    S-->>C: Session open
    C->>C: Assemble system prompt from rubric record

    loop Each exchange
        P->>C: Type a summary of what the team said
        C->>S: Send exchange
        S->>M: Stream request
        M-->>S: tokens
        S-->>C: SSE tokens
        C-->>P: Assessment renders as it is written
        Note over P: Proctor can intervene mid-stream —<br/>correct the input, add context, probe further
    end

    P->>C: Pass / No Pass + score (3 / 2 / 1 / 0)
    C->>S: Record result
    S->>D: Persist score
    S->>D: Store raw transcript (free, automatic)
    S-->>C: Coaching notes for this station
    C-->>P: Coaching panel expands
```

---

## 3. The Two-Lever Hold

Global hold and per-station hold are independent. Either blocks; the client is
told which.

```mermaid
stateDiagram-v2
    [*] --> Open

    Open --> EventHeld: admin holds all
    EventHeld --> Open: admin releases all

    Open --> StationHeld: admin holds one station
    StationHeld --> Open: admin releases that station

    EventHeld --> Both: admin also holds a station
    StationHeld --> Both: admin also holds all
    Both --> EventHeld: station released
    Both --> StationHeld: event released

    note right of Open
        New sessions permitted
    end note

    note right of Both
        Either lever blocks.
        Release of one does not
        imply release of the other.
    end note
```

**Why two levers.** A global pause is an event-level action — a briefing, a
schedule overrun. A station hold is a table-level action — the proctor stepped
away, the laptop died, the queue is being reorganised. One control would force
an operator to stop the entire circuit to deal with one table.

---

## 4. Attempt Accounting

```mermaid
flowchart TD
    START["Team requests a session"] --> HOLD{"Held?"}
    HOLD -->|yes| BLOCK["Refuse, state which lever"]
    HOLD -->|no| LIMIT{"Attempts used<br/>&lt; max for this station?"}
    LIMIT -->|no| DENY["Refuse: attempt limit reached"]
    LIMIT -->|yes| INC["INCREMENT ATTEMPT NOW"]
    INC --> OPEN["Session opens"]
    OPEN --> OUTCOME{"Outcome"}
    OUTCOME -->|verdict recorded| SCORE["Persist score 3/2/1/0"]
    OUTCOME -->|abandoned| NOTHING["Attempt already counted"]
    OUTCOME -->|client crash| NOTHING
    SCORE --> TIE["Attempts also break<br/>leaderboard ties"]
```

> Incrementing at **commitment** rather than at **result** closes the walk-away
> loophole and survives a client crash. The cost — a genuine technical failure
> consumes an attempt — is absorbed by an admin clear.

---

## 5. Multi-Tenant Scoping

```mermaid
graph TB
    subgraph Routes["Route Shape"]
        R1["/api/evaluations/*<br/>tenant management"]
        R2["/api/eval/:evalId/state"]
        R3["/api/eval/:evalId/result/*"]
        R4["/api/eval/:evalId/transcript/*"]
        R5["/api/eval/:evalId/usage"]
    end

    subgraph T1["Evaluation A"]
        A1["Rubric + stations"]
        A2["Team results"]
        A3["Transcripts"]
        A4["Usage totals"]
    end

    subgraph T2["Evaluation B"]
        B1["Rubric + stations"]
        B2["Team results"]
        B3["Transcripts"]
        B4["Usage totals"]
    end

    R2 & R3 & R4 & R5 -->|":evalId scopes<br/>every read and write"| T1
    R2 & R3 & R4 & R5 --> T2
    T1 -.->|"never"| T2
```

Scoping in the **route** rather than in handler logic means a newly added
endpoint cannot accidentally be written unscoped — the id is not available
except through the scoped path.

### Selection behaviour

```mermaid
flowchart LR
    LOAD["Client loads"] --> N{"Active evaluations"}
    N -->|exactly one| AUTO["Auto-select.<br/>Single-event experience unchanged."]
    N -->|more than one| PICK["Evaluation picker"]
    N -->|zero, user| WAIT["'Check back shortly'"]
    N -->|zero, admin| CREATE["Prompt to create one"]
    AUTO & PICK --> APP["Application"]
```

### Migration

```mermaid
flowchart TD
    BOOT["Service boots"] --> CHECK{"Any evaluation<br/>records exist?"}
    CHECK -->|yes| SKIP["No action"]
    CHECK -->|no| FOLD["Fold legacy single-config data<br/>into the first evaluation record"]
    FOLD --> TOUCH["Only update documents still<br/>missing the scoping field"]
    TOUCH --> DONE["Idempotent — safe to re-run,<br/>safe to redeploy"]
```

---

## 6. Free Capture, Paid Refinement

```mermaid
flowchart LR
    VERDICT["Verdict recorded"] -->|automatic, no model call| RAW[("Raw transcript stored")]
    RAW --> LATER{"Admin requests<br/>a scorecard?"}
    LATER -->|no| END["Nothing spent.<br/>Raw material retained."]
    LATER -->|yes| GEN["Model call: compose scorecard"]
    GEN --> METER["Record tokens + cost<br/>under 'scorecard' call kind"]
    METER --> PDFDOC["PDF generated and stored"]
    PDFDOC --> DL["Participant may download"]
    PDFDOC -.->|"force rebuild —<br/>explicit, costs again"| GEN
```

The expensive artifact is wanted for a minority of sessions; the raw material is
needed for all of them and is unrecoverable if not captured when it exists.
Metering by call kind is what makes the split verifiable rather than merely
intended.

---

## 7. Authentication and Role Resolution

```mermaid
sequenceDiagram
    participant B as Browser
    participant S as Server
    participant G as Identity Gateway

    B->>S: POST credentials
    S->>G: Delegate authentication
    G-->>S: Token + identity + role
    S->>S: Sign payload (HMAC) into httpOnly cookie
    S-->>B: Set-Cookie, redirect to app

    B->>S: GET / (application shell)
    S->>S: requireAuth — valid signed cookie?
    alt no valid cookie
        S-->>B: Redirect to login (JSON request gets 401)
    end
    S-->>B: Serve shell

    B->>S: GET admin-only data
    S->>S: requireAdmin — role check
    alt not admin
        S-->>B: 403
    end
    S-->>B: Data
```

No local password store and no session library. Static assets — including the
application shell — are served **behind** the auth middleware, so an
unauthenticated request never receives the client at all.
