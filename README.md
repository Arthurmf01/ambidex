# Ambidex

A voice-first personal operating system, built with AI agents and run by them.

I speak a thought into my phone. A few seconds later it is a line in a Markdown file. A background agent then picks it up, works out what I meant, and does the first 80% of it: drafts the message, researches the question, files the note. Every morning a second system reads everything and gives me one page that says what matters today.

I use it every day. This repository documents how it is built.

> **Why there is no source code here.** Ambidex runs on my real tasks, inbox, contacts and notes, and its tests and history are full of them. The code lives in private repositories. This repo is the architecture, the design decisions and a few sanitised excerpts, which is the part worth reading anyway.

## The shape of it

```mermaid
flowchart LR
  subgraph Phone
    PWA["Ambidex PWA<br/>one button, speak"]
    SC["iOS Shortcut<br/>share sheet"]
  end

  subgraph Edge["Cloudflare (free tier)"]
    CW["Capture Worker<br/>auth, transcribe, append"]
    AI["Workers AI<br/>Whisper large-v3-turbo"]
    BW["Brief Worker<br/>render, act, write back"]
    KV[("KV<br/>briefs, tickets, state")]
  end

  subgraph Truth["Source of truth"]
    GH[("Private Git repo<br/>Markdown notes and task files")]
  end

  subgraph Mac["Laptop (scheduled jobs)"]
    ING["Ingest and dispatch<br/>classify, mint ticket"]
    AG["Capture agent<br/>headless Claude Code"]
    GEN["Morning generator<br/>narrative, triage, clean-up"]
    EX["Executor and mail runner<br/>tool-locked agents"]
  end

  PWA --> CW
  SC --> CW
  CW --> AI
  CW -- "Contents API" --> GH
  GH --> ING --> AG
  AG -- "drafts, notes, closes line" --> GH
  GH --> GEN -- "daily payload" --> BW
  BW <--> KV
  BW -- "live read, tick, sort, archive" --> GH
  KV --> EX -- "results" --> KV
  GEN --> TG["Telegram<br/>one link each morning"]
```

Three loops share one store:

| Loop | What it does | Where it runs |
|---|---|---|
| **Capture** | Voice or share sheet to a task line, in seconds | Phone to Cloudflare Worker |
| **Act** | An agent reads each capture, classifies the intent, and produces a draft or a note for my review | Laptop, scheduled, headless Claude Code |
| **Review** | A daily page that surfaces what to do, what is going stale, and what the AI suggests pruning | Laptop generator plus Cloudflare Worker |

## Step by step: what happens to one thought

Follow a single capture from my voice to a finished task. Blue steps are automatic, orange steps are an AI agent working, green steps are me deciding.

```mermaid
flowchart TD
  A(["1 · I tap the button and speak"]):::human --> B["2 · Worker checks my secret<br/>and transcribes the audio"]:::auto
  B --> C{"3 · Text looks right?"}:::human
  C -- "no" --> C2["I fix it or re-record"]:::human --> C
  C -- "yes, save" --> D["4 · One raw line is appended<br/>to the Inbox of the task file"]:::auto
  D --> E{"5 · Is this something<br/>an agent can start?"}:::auto
  E -- "no: a plain to-do" --> P["Stays a normal task.<br/>No agent touches it"]:::auto
  E -- "yes" --> F["6 · A ticket is created:<br/>stable id, original words kept"]:::auto
  F --> G{"7 · Anything in the queue?<br/>(free check, no AI call)"}:::auto
  G -- "no" --> G2["Job exits.<br/>Nothing is spent"]:::auto
  G -- "yes" --> H["8 · Agent takes over:<br/>reads the ticket, picks one intent"]:::agent
  H --> I{"9 · Is the intent clear?"}:::agent
  I -- "no" --> S["Agent does nothing.<br/>Line marked skipped,<br/>stays a normal task"]:::agent
  I -- "yes" --> J["10 · Agent does the first 80%:<br/>drafts the message, writes the note,<br/>prepares the email or calendar entry"]:::agent
  J --> K["11 · Result is saved as a draft<br/>and I get a Telegram ping"]:::auto
  K --> L{"12 · I review the draft"}:::human
  L -- "good" --> L1["I send or use it myself"]:::human
  L -- "not good" --> L2["I edit or discard it.<br/>Nothing was sent"]:::human
  P --> M["13 · Next morning the brief<br/>lists everything still open"]:::auto
  S --> M
  L1 --> M
  L2 --> M
  M --> N{"14 · Does the task<br/>need a decision?"}:::auto
  N -- "no: sorted and on time" --> N1["Shown in the day's list"]:::auto
  N -- "yes: unsorted, stale,<br/>or AI suggests a clean-up" --> O{"15 · My call, one tap"}:::human
  O -- "sort" --> O1["Filed under a life objective"]:::human
  O -- "done" --> O2["Moved to the archive"]:::human
  O -- "won't do" --> O3["Moved to the archive<br/>as dropped"]:::human
  O -- "keep" --> O4["Hidden for two weeks"]:::human
  O2 --> R["Visible in the Done view<br/>for 30 days, can be reopened"]:::auto
  O3 --> R

  classDef auto fill:#e8f1fd,stroke:#3b82f6,color:#0b1f3a
  classDef agent fill:#fff1e0,stroke:#f59e0b,color:#3a2400
  classDef human fill:#e7f8ee,stroke:#22c55e,color:#0b2e1a
```

The two rules that keep this safe:

- **An agent only acts when the intent is clear.** If it is not, the capture stays an ordinary task and waits for me.
- **An agent never completes the outward-facing step.** It drafts; sending, deleting and deciding stay with me.

## What makes it interesting

**1. A Markdown file is the database.** Tasks live as lines in a Markdown file in a private Git repository. The phone, the Workers and every agent read and write that one file through the GitHub Contents API. No app database, no sync layer, no lock-in. Git gives history, undo and an audit trail of every change an agent has ever made.

**2. The AI proposes, a human disposes.** Agents draft; they never send. The weekly clean-up assistant is read-only: it proposes "drop", "already done" or "file under objective X", and each proposal becomes a button I tap or dismiss. Nothing is deleted, ever. Completed and dropped tasks move to an archive and can be reopened.

**3. Model output is never trusted directly.** Every structured answer from a model is validated against the real data before it can reach a button: unknown ids, unknown actions and impossible combinations are discarded. A hallucinated task id cannot become a destructive action. See [patterns](docs/patterns.md).

**4. Each agent gets the smallest toolbox that does its job.** The executor that attempts tasks has a file-and-search allow-list and no shell. The mail triage agent has no tools at all and only writes text; a small API client does the I/O, and its send function requires an approval token that only a verified human tap can create. The guest agent runs in a copied directory that contains nothing personal.

**5. Everything fails soft.** If the model times out, the brief still renders from the last good pass. If a data source is unreachable, that section is omitted and the rest ships. A missing age is treated as "unknown", never guessed as "stale".

**6. It costs nothing to run.** Cloudflare free tier for compute, storage and speech-to-text. Agent work runs through headless Claude Code on a flat subscription, so there is no per-call API bill. The brief shows what the same usage would have cost at API list prices.

## Components

| Component | Stack | Role |
|---|---|---|
| Ambidex PWA | Single HTML file, vanilla JS, `MediaRecorder`, Web Audio | One-button voice capture with live waveform, type-instead fallback, photo attach |
| Capture Worker | TypeScript, Cloudflare Workers, Workers AI | Auth, transcription, append to the Inbox with retry on write conflicts |
| Ingest and dispatch | Python, launchd | Classifies each capture, mints a ticket with a stable id, queues it for an agent |
| Capture agent | Headless Claude Code driven by a skill file | More than a dozen intents, including outreach draft, people sourcing, research note, quick lookup, link gist, brain-dump to note, calendar entry and email draft |
| Morning generator | TypeScript on Node, launchd | Reads notes, tasks and agent transcripts; writes the narrative; triages; runs the weekly clean-up pass |
| Brief Worker | TypeScript, Cloudflare Workers, KV | Renders the page, reads the task file live, writes ticks, sorts, drops, reopens and archives |
| Executor and mail runner | TypeScript, headless Claude Code, launchd | Drains tickets opened from the page, under tool allow-lists |

## Read more

- [Architecture](docs/architecture.md): data flows, the ticket lifecycle, the review logic
- [Design decisions](docs/design-decisions.md): the choices that held, and the ones that were wrong
- [Patterns](docs/patterns.md): sanitised code excerpts for the ideas above
- [How it was built](docs/how-it-was-built.md): the spec-first, agent-built workflow behind it

## Numbers

| | |
|---|---|
| Automated tests | 368 on the brief, 92 on the capture Worker, 47 on ingest and dispatch |
| Running cost | Zero infrastructure, one flat AI subscription |
| Time from speaking to a saved task | Seconds |
| Built | August to October 2026, alongside a full-time MBA |

## About

Built by [Arthur Maroquene Froissart](https://www.linkedin.com/in/arthurmf/). Six years in enterprise software at Salesforce and MuleSoft, now an INSEAD MBA building with AI. The planning and build workflow used here is open in [claude-skills](https://github.com/Arthurmf01/claude-skills).
