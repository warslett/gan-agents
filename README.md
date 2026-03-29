# GAN Agents

A multi-agent system that applies the Generative Adversarial Network (GAN) concept to creative problem-solving using Claude sub-agents.

## Concept

In machine learning, a [GAN](https://aws.amazon.com/what-is/gan/) consists of two competing neural networks: a **Generator** that creates outputs and a **Discriminator** that evaluates them. The adversarial tension between them drives both to improve until the Generator produces outputs that can withstand the Discriminator's scrutiny.

This project applies the same principle to idea generation. Instead of neural networks, it uses **Claude sub-agents** organised into adversarial teams:

- **Generator agents** propose and refine creative ideas.
- **Discriminator agents** critically evaluate and challenge those ideas.
- A **neutral Orchestrator** mediates between them, ensuring neither side can influence the other.

Ideas are iteratively strengthened through multiple rounds of generation, criticism, and revision — with the weakest idea eliminated each round — until a single, battle-tested idea survives.

## Usage

**As a plugin:**

```
/gan:generate Give me an idea for a new startup
```

**As a standalone skill (when working inside this repo):**

```
/gan Give me an idea for a new startup
```

### Options

| Flag | Description | Default |
|---|---|---|
| `--ideas N` | Number of ideas to generate (minimum 2). Controls the number of rounds: N ideas = N-1 eviction rounds. | 5 |
| `--dir PATH` | Directory to use for problem files. Can be absolute or relative. If omitted, a directory is created automatically under `problems/` based on the problem statement. | Auto-generated |

Options can appear anywhere in the arguments:

```
/gan:generate --ideas 3 Give me an idea for a new startup
/gan:generate Give me an idea for a new startup --dir problems/my-startup
/gan:generate --ideas 8 --dir /tmp/experiment What should I build next?
```

Fewer ideas means a faster, cheaper run. More ideas means broader exploration and more rounds of refinement.

### Example Problems

This tool is problem-agnostic. Some examples:

- ["Give me an idea for a new startup in the education space"](examples/new-education-startup/summary.md)
- ["Where should I go travelling this year if I enjoy hiking and history?"](examples/hiking-and-history-travel/summary.md)
- ["How should I architect a real-time multiplayer game backend?"](examples/realtime-multiplayer-game-backend/summary.md)
- ["What would be a good premise for a new science fiction novel?"](examples/sci-fi-novel-premise/summary.md)
- "What new car should I buy if I prioritise safety and fuel efficiency?"
- "How should I invest $50,000 for long-term growth?"

### What to Expect

- After invoking the skill, you will be asked how you want to score ideas (defaults, select from list, or define your own). If you define your own dimensions, you will be asked a short clarifying question for each one.
- The process involves many agent invocations and will take several minutes to complete.
- You can follow progress by watching the files appear in the problem directory.
- The final `summary.md` will be presented to you when the process completes.
- All intermediate drafts and criticism documents are preserved for transparency.

## Architecture

### Agents

| Agent | Role | Invocation |
|---|---|---|
| **GAN Orchestrator** | Neutral mediator. Manages the process, creates files, executes evictions. | Entry point — invoked by the user |
| **Generator Orchestrator** | Manages Generator Workers. Handles ideation (round 1) and revision (rounds 2+). | Invoked by GAN Orchestrator |
| **Generator Ideation Worker** | Creative blue-sky thinker. Proposes bold, original ideas with research. | Invoked by Generator Orchestrator (round 1 only) |
| **Generator Revision Worker** | Idea advocate. Strengthens and defends an idea against specific criticism. | Invoked by Generator Orchestrator (revision rounds) |
| **Discriminator Orchestrator** | Manages Discriminator Workers. Aggregates scores and recommends eviction. | Invoked by GAN Orchestrator |
| **Discriminator Worker** | Adversarial critic. Scores ideas on user-selected dimensions and exposes weaknesses. | Invoked by Discriminator Orchestrator |

### Adversarial Boundary

The Generator and Discriminator sides **never communicate directly**. The GAN Orchestrator is the sole mediator:

```
                    GAN Orchestrator
                   /                \
    Generator Orchestrator    Discriminator Orchestrator
      /    |    \                /    |    \
   Worker Worker Worker     Worker Worker Worker
```

- The Generator side only sees **criticism documents** — never the Discriminator's internal reasoning.
- The Discriminator side only sees **draft documents** — never the Generator's internal reasoning.
- All communication flows through **markdown files** in the problem directory.

This separation ensures the adversarial dynamic is maintained — neither side can game the other.

### Context Isolation

All sub-agents are invoked with **fresh context** every time. No state carries over between invocations. The markdown files in the problem directory are the sole source of persistent state. This prevents accumulated bias and ensures each round is evaluated on its merits.

## The Process

### Round 1 — Ideation

1. The GAN Orchestrator creates (or uses) the problem directory.
2. The Generator Orchestrator invokes **K Ideation Workers sequentially** (where K is the number of ideas, default 5). Each worker:
   - Receives the problem statement.
   - Gets one-sentence summaries of previously proposed ideas (to ensure diversity).
   - Performs web research for inspiration.
   - Returns a detailed, original idea.
3. The Generator Orchestrator compiles all K ideas into `draft_1.md`.

### Rounds 1 to K-1 — Criticism & Eviction

For each round N (from 1 to K-1):

4. The Discriminator Orchestrator reads `draft_N.md` and invokes **one Discriminator Worker per idea, in parallel**. Each worker:
   - Receives only the single idea it must evaluate.
   - Performs web research to fact-check and challenge the idea.
   - Scores the idea on the **user-selected dimensions** (1–10 each).
   - Provides detailed prose criticism.
   - Gives a verdict: **Strong**, **Weak**, or **Evict**.
5. The Discriminator Orchestrator aggregates scores and recommends which idea to **evict** (the one with the lowest aggregate score).
6. The criticism is written to `draft_N_criticism.md`.
7. The GAN Orchestrator **executes the eviction** — it always follows the Discriminator's recommendation.

### Rounds 2 to K — Revision

8. The Generator Orchestrator reads the criticism and invokes **one Revision Worker per remaining idea, in parallel**. Each worker:
   - Receives only its own idea and its own criticism (no cross-idea information).
   - Defends the idea against unfair criticism with evidence.
   - Addresses valid criticism by revising the idea.
   - Uses web research only to find supporting evidence or verify facts (not to reimagine the idea).
9. The revised ideas are compiled into `draft_[N+1].md`.

### Summary

10. The GAN Orchestrator reads all drafts and criticism files, then produces `summary.md` containing:
     - The original problem.
     - All K original ideas and how they evolved.
     - Which ideas were evicted, when, and why.
     - The final surviving idea and why it won.
     - Score progressions across rounds.

## Scoring Dimensions

Before the GAN process begins, the user chooses how ideas will be scored. There are three options:

1. **Use defaults** — all 8 predefined dimensions are used.
2. **Select from list** — the user picks a subset of the 8 predefined dimensions.
3. **Define my own** — the user provides custom dimension names. Each custom dimension is then clarified through a follow-up question with suggested interpretations, so the Discriminator knows exactly what to assess.

This means scoring is tailored to the problem. A question about holiday destinations might only need Cost, Fun, and Safety, while a startup idea might benefit from Feasibility, Impact, and Rigour.

### Predefined Dimensions

| Dimension | What It Measures |
|---|---|
| **Feasibility** | Can this realistically be done with available resources and technology? |
| **Originality** | Is this genuinely novel or a rehash of existing ideas? |
| **Impact** | How valuable is the outcome if it works? |
| **Simplicity** | Is the idea elegant, easy to understand, and free of unnecessary complexity? |
| **Risk Management** | How well does the idea anticipate, mitigate, and manage risks? |
| **Rigour** | Is the reasoning well-founded and thorough, or mostly speculation? |
| **Coherence** | Does the idea hold together logically without contradictions? |
| **Adaptability** | How robust is the idea to changing circumstances or assumptions? |

The **aggregate score** is the mean of all selected dimensions. Eviction is based on the lowest aggregate score, with tiebreakers on verdict count and then the first dimension in the list.

## File Structure

After a complete run with K ideas, the problem directory will contain:

```
problems/
  your-problem-slug/
    draft_1.md                  # K initial ideas
    draft_1_criticism.md        # Criticism of all K, evicts 1
    draft_2.md                  # K-1 revised ideas
    draft_2_criticism.md        # Criticism of K-1, evicts 1
    ...                         # (one eviction per round)
    draft_K.md                  # 1 surviving idea (final revision)
    summary.md                  # Complete process summary
```

With the default of 5 ideas, this produces `draft_1.md` through `draft_5.md` and `draft_1_criticism.md` through `draft_4_criticism.md`.

## Installation

### As a plugin (recommended)

The `gan-plugin/` directory is a ready-to-use Claude Code plugin. You can install it in several ways:

**Local testing:**

```bash
claude --plugin-dir ./gan-plugin
```

**Install from a local path** (persists across sessions):

```
/plugin install --path ./gan-plugin
```

**Install from a Git repository:**

```
/plugin install gan@https://github.com/warslett/gan-agents
```

Once installed, the skill is available as `/gan:generate`.

### As a standalone project skill

If you clone this repo and work inside it, the skill and agents under `.claude/` are available directly as `/gan`. This is useful for development and experimentation.

## Design Principles

1. **Adversarial tension drives quality.** The Generator and Discriminator have opposing objectives — this tension produces ideas that are both creative and resilient.
2. **Separation of concerns.** The Generator never evaluates; the Discriminator never creates; the Orchestrator never influences. Each role is distinct.
3. **Evidence over opinion.** Both sides use web research to ground their arguments in evidence, not just reasoning.
4. **Transparency.** Every draft, criticism, score, and eviction decision is documented in markdown files. The entire process is auditable.
5. **Fresh context, clean thinking.** Agents start fresh each round, preventing accumulated bias and ensuring each evaluation is independent.
