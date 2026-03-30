# GAN Agents

A multi-agent system that applies the Generative Adversarial Network (GAN) concept to creative problem-solving using Claude sub-agents.

## Concept

In machine learning, a [GAN](https://aws.amazon.com/what-is/gan/) consists of two competing neural networks: a **Generator** that creates outputs and a **Discriminator** that evaluates them. The adversarial tension between them drives both to improve until the Generator produces outputs that can withstand the Discriminator's scrutiny.

This project applies the same principle to idea generation. Instead of neural networks, it uses **Claude sub-agents** organised into adversarial teams:

- **Generator agents** propose and refine creative ideas.
- **Discriminator agents** critically evaluate and challenge those ideas.
- A **neutral orchestration skill** mediates between them, ensuring neither side can influence the other.

Ideas are iteratively strengthened through multiple rounds of generation, criticism, and revision — with the weakest idea eliminated each round — until a single, battle-tested idea survives.

## Usage

```
/gan:generate Give me an idea for a new startup
```

### Options

| Flag | Description | Default |
|---|---|---|
| `--dir PATH` | Directory to use for problem files. Can be absolute or relative. If omitted, a directory is created automatically under `problems/` based on the problem statement. | Auto-generated |

```
/gan:generate Give me an idea for a new startup
/gan:generate Give me an idea for a new startup --dir problems/my-startup
```

After invoking the skill, you will be asked interactively how many ideas to generate (3, 5, 10, or a custom number), how to score them, and whether to perform web research. Fewer ideas means a faster, cheaper run. More ideas means broader exploration and more rounds of refinement.

### Example Problems

This tool is problem-agnostic. Some examples:

- ["Give me some fun ideas for a scout troop leader to organise for their scout troop. The troop has girls and boys aged 10 - 14"](examples/scout-troop-activities/summary.md)
- ["Where should I go travelling this year for one week in Europe if I enjoy hiking and history but don't like very hot weather?"](examples/europe-hiking-history-trip/summary.md)
- ["What software architecture should I use for a virtual rummikub game allowing multiple players to play against each other or against an AI over the web?"](examples/virtual-rummikub-game-architecture/summary.md)
- ["Give me an idea for a new startup in the education space"](examples/new-education-startup/summary.md)


### What to Expect

- After invoking the skill, you will be guided through an interactive setup: how many ideas (3, 5, 10, or custom), how to score them (choose from suggested dimension sets or define your own), and whether to perform web research (yes, no, or custom instructions). If you define your own scoring dimensions, you will be asked a short clarifying question for each one.
- The process involves many agent invocations and will take several minutes to complete.
- You can follow progress by watching the files appear in the problem directory.
- A short summary will be displayed when the process completes. The full `summary.md` is written to the problem directory.
- All intermediate ideas, criticisms, and research are preserved as individual files for transparency.

## Architecture

### Skill and Agents

The system uses a **skill** as the orchestrator and **worker sub-agents** for the creative and evaluative work. The skill runs in the main conversation context and spawns workers as sub-agents — this flat architecture is required because Claude Code sub-agents cannot spawn other sub-agents.

| Component | Role | Type |
|---|---|---|
| **Generate Skill** | Lightweight orchestrator. Creates directories, passes file paths, triggers agents. Never reads or writes problem files. | Skill (runs in main conversation) |
| **Research Worker** | Finds information-rich online resources and extracts useful content into a research library. Optional — user can skip or customise. | Sub-agent (before ideation, if enabled) |
| **Generator Ideation Worker** | Creative blue-sky thinker. Reads research, proposes bold, original ideas. Writes its idea to a file. | Sub-agent (round 1 only) |
| **Generator Revision Worker** | Idea advocate. Reads its idea and criticism files, strengthens the idea by addressing valid criticism — simplifying where possible rather than adding complexity. | Sub-agent (revision rounds) |
| **Discriminator Worker** | Adversarial critic. Reads an idea file, scores it per dimension with detailed feedback organised by dimension, diagnosing weaknesses without prescribing fixes. | Sub-agent (each criticism round) |
| **Discriminator Judge** | Impartial arbiter. Reads all criticism files for a round, returns the eviction decision. | Sub-agent (each criticism round) |
| **Summary Writer** | Reads all files across all rounds and writes a comprehensive summary document. | Sub-agent (final step) |

### File-Path Orchestration

The skill operates as a lightweight coordinator that only passes file paths between agents — it never reads or writes problem content itself. This "pass by reference" approach keeps the main conversation context minimal regardless of how many ideas or rounds are involved.

### Adversarial Boundary

The Generator and Discriminator sides **never communicate directly**. The skill is the sole mediator:

```
                         Generate Skill (orchestrator)
                /       |        |          |         \
   Research Worker  Ideation  Revision  Disc. Workers  Disc. Judge
```

- The Generator side only sees **criticism files** — never the Discriminator's internal reasoning.
- The Discriminator side only sees **idea files** — never the Generator's internal reasoning.
- Each worker sees **only its own idea** — never other ideas or other workers' evaluations.
- The Judge sees **only criticism files** — never the original ideas.
- All communication flows through **markdown files** in the problem directory.

This separation ensures the adversarial dynamic is maintained — neither side can game the other.

### Context Isolation

All sub-agents are invoked with **fresh context** every time. No state carries over between invocations. The markdown files in the problem directory are the sole source of persistent state. This prevents accumulated bias and ensures each round is evaluated on its merits.

## The Process

### Research (optional)

1. If enabled, the skill invokes a **Research Worker** which:
   - Searches the web (up to 3 searches) for information-rich resources — "top 10" lists, comparison articles, broad overviews.
   - Fetches up to 10 selected resources and extracts useful content (facts, options, data, comparisons), filtering out noise.
   - Writes an extracted-information file for each resource to `research/resource_N.md`.
   - Creates `research/index.md` linking to all resources with short descriptions.
   - The research worker gathers raw material — it does not propose or evaluate solutions.

### Round 1 — Ideation

2. The skill invokes **K Ideation Workers sequentially**. Each worker:
   - Receives the problem statement and an output file path. If research was performed, also receives the research index path.
   - Reads the research library (if available) for inspiration.
   - Gets one-sentence summaries of previously proposed ideas (to ensure diversity).
   - Writes its idea directly to `round_1/ideas/idea_N.md`.
   - Returns a one-sentence summary (for passing to subsequent workers).

### Rounds 1 to K-1 — Criticism & Eviction

For each round N (from 1 to K-1):

3. The skill creates `round_N/criticism/` and invokes **one Discriminator Worker per idea, in parallel**. Each worker:
   - Reads its assigned idea file from `round_N/ideas/`.
   - Scores the idea on the **user-selected dimensions** (1–10 each), only crediting claims that are concretely supported.
   - Writes criticism organised by dimension — diagnosing what is holding each score back without prescribing fixes.
   - Writes to `round_N/criticism/idea_X_criticism.md`.
4. The skill invokes a **Discriminator Judge** with the list of criticism file paths. The judge reads all criticisms and returns the eviction decision (lowest aggregate score).
5. The skill removes the evicted idea from the surviving list.

### Rounds 2 to K — Revision

6. The skill creates `round_[N+1]/ideas/` and invokes **one Revision Worker per surviving idea, in parallel**. Each worker:
   - Reads its own idea file and its own criticism file from the previous round.
   - Strengthens the idea by addressing valid criticism and defending against unfair criticism — preferring simplification over added complexity.
   - Writes the revised idea to `round_[N+1]/ideas/idea_X.md`.

### Round K — Final Criticism

7. After the last revision, one idea remains. The skill invokes a **single Discriminator Worker** on the surviving idea to produce a final score and document any remaining weaknesses. No eviction or revision follows.

### Summary

8. The skill invokes a **Summary Writer** with the problem directory path. The summary writer:
   - Reads all files across all rounds.
   - Writes `summary.md` with the complete process history, scores, and evolution.
   - Returns a short summary for display.

## Scoring Dimensions

Before the GAN process begins, the user chooses how ideas will be scored. The system analyses the problem statement and suggests **3 sets of 4 dimensions**, each representing a different evaluation lens. The user can pick a suggested set or type their own comma-separated list of custom dimensions. Custom dimensions are each clarified through a follow-up question with suggested interpretations, so the Discriminator knows exactly what to assess.

This means scoring is tailored to the problem. A question about software architecture might be scored on Feasibility, Simplicity, Scalability, and Security, while a creative writing prompt might use Originality, Emotional Impact, Coherence, and Practicality.

The **aggregate score** is the mean of all selected dimensions. Eviction is based on the lowest aggregate score, with tiebreakers determined by the severity and number of weaknesses identified in the critical analysis.

## File Structure

After a complete run with K ideas, the problem directory will contain:

```
problems/
  your-problem-slug/
    research/                     # (only if research was enabled)
      index.md                  # Links to all resources with descriptions
      resource_1.md             # Extracted information from an online resource
      resource_2.md
      ...
    round_1/
      ideas/
        idea_1.md               # Each idea in its own file
        idea_2.md
        idea_3.md
      criticism/
        idea_1_criticism.md     # Each criticism in its own file
        idea_2_criticism.md
        idea_3_criticism.md
    round_2/
      ideas/
        idea_1.md               # Revised surviving ideas
        idea_3.md               # (idea_2 was evicted)
      criticism/
        idea_1_criticism.md
        idea_3_criticism.md
    round_3/
      ideas/
        idea_1.md               # Final surviving idea
      criticism/
        idea_1_criticism.md     # Final criticism with score
    summary.md                  # Complete process summary
```

Each worker writes its own file directly. The skill never reads or writes problem content — it only creates directories and passes file paths. This keeps the main conversation context minimal regardless of how many ideas or rounds are involved.

## Installation

### As a plugin

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

## Design Principles

1. **Adversarial tension drives quality.** The Generator and Discriminator have opposing objectives — this tension produces ideas that are both creative and resilient.
2. **Separation of concerns.** The Generator never evaluates; the Discriminator never creates; the orchestrating skill never influences. Each role is distinct.
3. **Evidence over opinion.** An optional research step gathers information-rich online resources into a shared library. Ideation workers draw on this research to ground ideas in evidence. Discriminator and revision workers rely on reasoning and specifics.
4. **Transparency.** Every draft, criticism, score, and eviction decision is documented in markdown files. The entire process is auditable.
5. **Fresh context, clean thinking.** Agents start fresh each round, preventing accumulated bias and ensuring each evaluation is independent.
