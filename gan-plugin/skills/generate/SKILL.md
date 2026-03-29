---
description: Run the GAN-inspired adversarial idea generation process. Invoke with a problem statement to generate, critique, and refine ideas through multiple rounds. Options: --ideas N (default 5, minimum 2), --dir PATH (problem directory).
user_invocable: true
---

1. Parse the arguments. Extract and remove any optional flags from the problem statement:
   - `--ideas N` (where N is a number, minimum 2): the number of ideas to generate. Default: 5.
   - `--dir PATH`: the directory to use for problem files. PATH can be absolute or relative to the current working directory.
2. If `--dir` was provided, use that path as the problem directory. Otherwise, create a directory under `problems/` using a short, lowercase, hyphen-separated slug based on the problem (e.g., `problems/new-startup-idea/`). In either case, use `mkdir -p` via the Bash tool to ensure the directory exists.
3. Ask the user how they want to score ideas. Use the `AskUserQuestion` tool with a SINGLE question (multiSelect: false):

   **Question** (header: "Scoring"):
   > "How do you want to score ideas?"
   - **Use defaults** — "Use all 8 predefined scoring dimensions (Feasibility, Originality, Impact, Simplicity, Risk Management, Rigour, Coherence, Adaptability)"
   - **Select from list** — "Choose which of the 8 predefined dimensions to include"
   - **Define my own** — "Specify your own custom scoring dimensions"

   Then proceed based on the user's choice:

   ### Option A: "Use defaults"

   Use all 8 predefined dimensions: Feasibility, Originality, Impact, Simplicity, Risk Management, Rigour, Coherence, Adaptability.

   ### Option B: "Select from list"

   Use the `AskUserQuestion` tool with TWO multiSelect questions. The user selects the dimensions they want. Present all 8 available dimensions split across 2 questions:

   **Question 1** (header: "Criteria 1/2"):
   > "Which scoring dimensions do you want to evaluate ideas against? (1/2)"
   - **Feasibility** — "Can this realistically be done? Are resources, technology, and skills available?"
   - **Originality** — "Is this genuinely novel, or a rehash of existing ideas?"
   - **Impact** — "If it works, how valuable is the outcome? Is the upside significant?"
   - **Simplicity** — "Is the idea elegant and easy to understand? Does it avoid unnecessary complexity?"

   **Question 2** (header: "Criteria 2/2"):
   > "Which scoring dimensions do you want to evaluate ideas against? (2/2)"
   - **Risk Management** — "How well does the idea anticipate and manage risks?"
   - **Rigour** — "Is the reasoning well-founded? Are claims backed by evidence?"
   - **Coherence** — "Does the idea hold together logically? Are there contradictions or gaps?"
   - **Adaptability** — "How robust is the idea to changing circumstances or assumptions?"

   Collect the user's selections from both questions. If the user selects fewer than 2 dimensions total, tell them at least 2 are required and ask again. Combine all selected dimensions into a single list.

   ### Option C: "Define my own"

   Use the `AskUserQuestion` tool (multiSelect: false) to ask the user to list their dimensions:

   **Question** (header: "Dimensions"):
   > "What dimensions do you want to score ideas against? List them separated by commas."
   - **Example: Cost, Fun, Novelty** — "Comma-separated list of dimension names"
   - **Example: Practicality, Wow Factor, Sustainability, Inclusivity** — "Comma-separated list of dimension names"

   The user will either select an example or type their own list via "Other". Parse their response into individual dimension names.

   Then, for EVERY dimension (even if its name matches a predefined dimension), ask a clarifying question using `AskUserQuestion` (multiSelect: false) to confirm what it means and how to score it. Do NOT skip any dimension — the user's intended meaning may differ from the predefined definition. Generate 2 plausible interpretations of the dimension as options based on the dimension name and the problem context. For example, if the user listed "Fun" for a children's activity problem:

   **Question** (header: "[Dimension]"):
   > "What should '[Dimension]' measure when scoring ideas?"
   - Option 1: a plausible interpretation (e.g., "How entertaining is this for a child on a day-to-day basis?")
   - Option 2: a different plausible interpretation (e.g., "How much excitement and engagement does the overall concept generate?")
   - Option 3: a different plausible interpretation (e.g., "How favourably would a child rate this compared to screen time or other default activities?")

   The user can select one of the 3 suggestions or type their own via "Other". Use their answer as the scoring description for that dimension.

   After clarifying all dimensions, compile them into the final list. Each dimension should have a name and a one-sentence description of what to assess (matching the format used by the predefined dimensions).

4. Invoke the `gan-orchestrator` agent with the user's problem statement, the number of ideas, the path to the problem directory, and the selected scoring dimensions:

> **Problem:** [the problem statement with all flags removed]
> **Number of ideas:** [N]
> **Problem directory:** [the problem directory path]
> **Scoring dimensions:** [list of selected dimensions — for predefined dimensions use the name only (e.g. "Feasibility, Impact"); for custom dimensions include the description (e.g. "Fun — How entertaining is this for a child on a day-to-day basis?")]
