# email-digest-scheduler

A personal tool that runs on my Mac every morning. It reads my job-search Gmail inbox (read-only), sorts replies
to my job applications with a **small model running locally**, and writes a short "where do I stand" summary
to a local folder. No email content leaves the computer.

This repo only hosts the [homepage](https://movchaneric.github.io/email-digest-scheduler/) and
[privacy policy](https://movchaneric.github.io/email-digest-scheduler/privacy.html) that Google's OAuth consent screen requires.

## How a daily run works

Purple is the only AI step. Everything else is plain code and fixed rules.

```mermaid
flowchart TD
    classDef ai fill:#f3e8ff,stroke:#9333ea,stroke-width:2px,color:#3b0764
    classDef code fill:#e0f2fe,stroke:#0284c7,color:#0c4a6e
    classDef ext fill:#fef3c7,stroke:#d97706,color:#78350f
    classDef store fill:#dcfce7,stroke:#16a34a,color:#14532d
    classDef fail fill:#fee2e2,stroke:#dc2626,color:#7f1d1d

    A["1 - launchd starts the run<br/>every day at 08:00<br/>(or when the Mac wakes)"]:::ext --> B
    B["2 - Check Ollama is running<br/>and the model is pulled"]:::code --> C
    C["3 - Load the Ignored Senders list<br/>and the saved state"]:::code --> D
    S[("state.json<br/>checkpoint, processed ids,<br/>Replies, Applications")]:::store -.-> C
    D["4 - Ask Gmail for mail since the checkpoint<br/>read-only, never writes"]:::ext --> E
    E["5 - Drop ids already processed<br/>(no duplicates)"]:::code --> F{"More emails<br/>to do?"}

    F -- yes --> G["6 - Fetch one email<br/>(subject, sender, longer of text/HTML body)"]:::ext --> H
    H{"7 - Pre-filter by rule:<br/>Ignored Sender or self-sent?"}:::code
    H -- "yes: Noise" --> M
    H -- no --> I

    I["8 - LOCAL MODEL qwen3:8b via Ollama<br/>one call per email, fixed JSON schema<br/>kind: noise / reply / application_sent<br/>verdict: positive / rejection / acknowledgement / unclear<br/>company, job title"]:::ai --> J
    J{"Valid answer?"}:::code
    J -- "no: skip, retry on next run" --> F
    J -- yes --> K

    K["9 - Match to an Application<br/>by company + job title<br/>(new Application if none)<br/>model said noise: nothing recorded"]:::code --> L
    L["10 - Record the Reply<br/>(same-day duplicates count once)"]:::code --> M
    M["11 - Mark id processed, move checkpoint,<br/>save state after every email"]:::code --> S
    M --> F

    F -- no --> N["12 - Work out each Application's Status<br/>Waiting, Silent after 30 days, Positive,<br/>Rejected, Unclear"]:::code --> O
    O["13 - Write the Digest<br/>YYYY-MM-DD.md, .html and latest.html"]:::store --> P
    P["14 - Remember when this Digest was written"]:::code --> Q
    Q["15 - Notification: N new Replies<br/>click opens latest.html"]:::ext

    B -. "any error, e.g. sign-in expired" .-> X["Digest failed: reason<br/>notification; progress is kept"]:::fail
    D -.-> X

    subgraph Legend
        direction LR
        L1["AI step (local model)"]:::ai
        L2["Plain code, rules"]:::code
        L3["Outside system"]:::ext
        L4["Files on the Mac"]:::store
    end
```

## Where the AI is (and isn't)

| Step | Who does it |
|---|---|
| Deciding what each email is (reply / application sent / noise), its verdict, company and job title | **Local model** (qwen3:8b via Ollama), one call per email, answer forced into a fixed JSON shape |
| Starting the run, reading Gmail, skipping already-seen emails | Code (launchd + Gmail API, read-only) |
| Dropping Ignored Senders and self-sent mail | Code (rules, before the model is asked) |
| Grouping Replies into Applications, working out each Status, counting | Code (rules) |
| Writing the Digest and sending the notification | Code |

It is **not an autonomous agent**: the model never chooses tools or next steps and can't act on anything. The
code runs the same fixed pipeline every day and asks the model one narrow question per email. That keeps it
predictable, cheap to run and safe with a mailbox.
