# GAN Agents

A multi-agent system that applies the Generative Adversarial Network (GAN) concept to creative problem-solving using Claude sub-agents.

## Concept

In machine learning, a [GAN](https://aws.amazon.com/what-is/gan/) consists of two competing neural networks: a **Generator** that creates outputs and a **Discriminator** that evaluates them. The adversarial tension between them drives both to improve until the Generator produces outputs that can withstand the Discriminator's scrutiny.

This project applies the same principle to idea generation. Instead of neural networks, it uses **Claude sub-agents** organised into adversarial teams:

- **Generator agents** propose and refine creative ideas.
- **Discriminator agents** critically evaluate and challenge those ideas.
- A **neutral Orchestrator** mediates between them, ensuring neither side can influence the other.

Ideas are iteratively strengthened through multiple rounds of generation, criticism, and revision — with the weakest idea eliminated each round — until a single, battle-tested idea survives.

## Architecture

### Agents

| Agent | Role | Invocation |
|---|---|---|
| **GAN Orchestrator** | Neutral mediator. Manages the process, creates files, executes evictions. | Entry point — invoked by the user |
| **Generator Orchestrator** | Manages Generator Workers. Handles ideation (round 1) and revision (rounds 2+). | Invoked by GAN Orchestrator |
| **Generator Ideation Worker** | Creative blue-sky thinker. Proposes bold, original ideas with research. | Invoked by Generator Orchestrator (round 1 only) |
| **Generator Revision Worker** | Idea advocate. Strengthens and defends an idea against specific criticism. | Invoked by Generator Orchestrator (revision rounds) |
| **Discriminator Orchestrator** | Manages Discriminator Workers. Aggregates scores and recommends eviction. | Invoked by GAN Orchestrator |
| **Discriminator Worker** | Adversarial critic. Scores ideas on 7 dimensions and exposes weaknesses. | Invoked by Discriminator Orchestrator |

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

1. The GAN Orchestrator creates a problem directory under `problems/`.
2. The Generator Orchestrator invokes **5 Ideation Workers sequentially**. Each worker:
   - Receives the problem statement.
   - Gets one-sentence summaries of previously proposed ideas (to ensure diversity).
   - Performs web research for inspiration.
   - Returns a detailed, original idea.
3. The Generator Orchestrator compiles all 5 ideas into `draft_1.md`.

### Rounds 1–4 — Criticism & Eviction

For each round N (from 1 to 4):

4. The Discriminator Orchestrator reads `draft_N.md` and invokes **one Discriminator Worker per idea, in parallel**. Each worker:
   - Receives only the single idea it must evaluate.
   - Performs web research to fact-check and challenge the idea.
   - Scores the idea on **7 dimensions** (1–10 each).
   - Provides detailed prose criticism.
   - Gives a verdict: **Strong**, **Weak**, or **Evict**.
5. The Discriminator Orchestrator aggregates scores and recommends which idea to **evict** (the one with the lowest aggregate score).
6. The criticism is written to `draft_N_criticism.md`.
7. The GAN Orchestrator **executes the eviction** — it always follows the Discriminator's recommendation.

### Rounds 2–5 — Revision

8. The Generator Orchestrator reads the criticism and invokes **one Revision Worker per remaining idea, in parallel**. Each worker:
   - Receives only its own idea and its own criticism (no cross-idea information).
   - Defends the idea against unfair criticism with evidence.
   - Addresses valid criticism by revising the idea.
   - Uses web research only to find supporting evidence or verify facts (not to reimagine the idea).
9. The revised ideas are compiled into `draft_[N+1].md`.

### Final Challenge & Revision

10. After `draft_5.md` (containing the single surviving idea), the Discriminator performs one **final challenge** — criticising the idea without an eviction recommendation.
11. The Generator performs one **final revision** based on this criticism, producing `draft_6.md`.

### Summary

12. The GAN Orchestrator reads all drafts and criticism files, then produces `summary.md` containing:
    - The original problem.
    - All 5 original ideas and how they evolved.
    - Which ideas were evicted, when, and why.
    - The final surviving idea and why it won.
    - Score progressions across rounds.

## Scoring Dimensions

Each idea is scored by Discriminator Workers on these 7 dimensions (1–10):

| Dimension | What It Measures |
|---|---|
| **Feasibility** | Can this realistically be done with available resources and technology? |
| **Originality** | Is this genuinely novel or a rehash of existing ideas? |
| **Risk of Failure** | How likely is this to go wrong? (Scored inversely — high risk = low score) |
| **Potential Impact or Benefit** | How valuable is the outcome if it works? |
| **Quality of Supporting Evidence** | Is the reasoning well-founded or speculative? |
| **Coherence** | Does the idea hold together logically without contradictions? |
| **Adaptability** | How robust is the idea to changing circumstances or assumptions? |

The **aggregate score** is the mean of all 7 dimensions. Eviction is based on the lowest aggregate score, with tiebreakers on verdict count and then feasibility.

## File Structure

After a complete run, the problem directory will contain:

```
problems/
  your-problem-slug/
    draft_1.md                  # 5 initial ideas
    draft_1_criticism.md        # Criticism of all 5, evicts 1
    draft_2.md                  # 4 revised ideas
    draft_2_criticism.md        # Criticism of 4, evicts 1
    draft_3.md                  # 3 revised ideas
    draft_3_criticism.md        # Criticism of 3, evicts 1
    draft_4.md                  # 2 revised ideas
    draft_4_criticism.md        # Criticism of 2, evicts 1
    draft_5.md                  # 1 surviving idea (revised)
    draft_5_criticism.md        # Final challenge (no eviction)
    draft_6.md                  # Final revision
    summary.md                  # Complete process summary
```

## Usage

Invoke the GAN Orchestrator agent with your problem:

```
/gan Give me an idea for a new startup
```

### Example Problems

This tool is problem-agnostic. Some examples:

- "Give me an idea for a new startup in the education space"
- "Where should I go travelling this year if I enjoy hiking and history?"
- "How should I architect a real-time multiplayer game backend?"
- "What would be a good premise for a new science fiction novel?"
- "What new car should I buy if I prioritise safety and fuel efficiency?"
- "How should I invest $50,000 for long-term growth?"

### What to Expect

- The process involves many agent invocations and will take several minutes to complete.
- You can follow progress by watching the files appear in the problem directory.
- The final `summary.md` will be presented to you when the process completes.
- All intermediate drafts and criticism documents are preserved for transparency.

## Design Principles

1. **Adversarial tension drives quality.** The Generator and Discriminator have opposing objectives — this tension produces ideas that are both creative and resilient.
2. **Separation of concerns.** The Generator never evaluates; the Discriminator never creates; the Orchestrator never influences. Each role is distinct.
3. **Evidence over opinion.** Both sides use web research to ground their arguments in evidence, not just reasoning.
4. **Transparency.** Every draft, criticism, score, and eviction decision is documented in markdown files. The entire process is auditable.
5. **Fresh context, clean thinking.** Agents start fresh each round, preventing accumulated bias and ensuring each evaluation is independent.
