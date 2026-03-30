---
description: Run the GAN-inspired adversarial idea generation process. Invoke with a problem statement to generate, critique, and refine ideas through multiple rounds. Options: --dir PATH (problem directory).
user_invocable: true
---

1. Parse the arguments. Extract and remove any optional flags from the problem statement:
   - `--dir PATH`: the directory to use for problem files. PATH can be absolute or relative to the current working directory.
2. If `--dir` was provided, use that path as the problem directory. Otherwise, create a directory under `problems/` using a short, lowercase, hyphen-separated slug based on the problem (e.g., `problems/new-startup-idea/`). In either case, use `mkdir -p` via the Bash tool to ensure the directory exists.
3. Ask the user how many ideas to generate. Use the `AskUserQuestion` tool with a SINGLE question (multiSelect: false):

   **Question** (header: "Ideas"):
   > "How many ideas should be generated?"
   - **3** — "Quick run — 3 ideas, 2 eviction rounds"
   - **5** — "Standard run — 5 ideas, 4 eviction rounds"
   - **10** — "Deep exploration — 10 ideas, 9 eviction rounds"

   The user may also type a custom number via "Other". Parse their response as the number of ideas (K). If K is less than 2, tell the user the minimum is 2 and ask again.

4. Ask the user how they want to score ideas. Use the `AskUserQuestion` tool with a SINGLE question (multiSelect: false):

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

   Collect the user's selections from both questions. If the user enters custom text via "Other" instead of selecting from the predefined options, disregard the custom text, explain that this step is for selecting from predefined dimensions only (and that they can go back and choose "Define my own" if they want custom dimensions), and re-ask the question. If the user selects fewer than 2 dimensions total, tell them at least 2 are required and ask again. Combine all selected dimensions into a single list.

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

5. **Research (optional).** Ask the user whether they want web research using `AskUserQuestion` (multiSelect: false):

   **Question** (header: "Research"):
   > "Should ideas be informed by web research?"
   - **Yes** — "Research up to 10 online resources before generating ideas"
   - **No** — "Skip research and generate ideas without web input"

   The user may also type their own instruction via "Other" (e.g. "Find 20 resources" or "Focus research on outdoor activities"). Handle each case:

   - **Yes:** Run `mkdir -p [problem_dir]/research` via the Bash tool. Invoke a `gan:research-worker` agent with the default prompt:
     > Research the following problem and compile a library of useful online resources.
     >
     > **Problem:** [the problem]
     > **Research directory:** [problem_dir]/research

   - **Custom text:** Run `mkdir -p [problem_dir]/research` via the Bash tool. Invoke a `gan:research-worker` agent, appending the user's custom instruction:
     > Research the following problem and compile a library of useful online resources.
     >
     > **Problem:** [the problem]
     > **Research directory:** [problem_dir]/research
     > **Additional instructions:** [the user's custom text]

   - **No:** Skip research entirely. No research worker is invoked and no research directory is created.

   If research was performed, store the path to `index.md` for use in step 6. If research was skipped, note that no research index is available.

6. **Round 1 — Ideation.**

   Run `mkdir -p [problem_dir]/round_1/ideas` via the Bash tool.

   Invoke `gan:generator-ideation-worker` agents **K times sequentially** using the Agent tool. Maintain a list of one-sentence idea summaries returned by each worker.

   If research was performed, include the research index path in each invocation. If research was skipped, omit the research index line.

   For the **first** invocation:
   > You are given a problem to solve. Come up with one creative, original idea.
   > **Problem:** [the problem]
   > **Research index:** [path to research/index.md] *(omit this line if research was skipped)*
   > **Output file:** [problem_dir]/round_1/ideas/idea_1.md

   For each **subsequent** invocation (idea N):
   > You are given a problem to solve. Come up with one creative, original idea that is distinctly different from the ideas already proposed.
   > **Problem:** [the problem]
   > **Research index:** [path to research/index.md] *(omit this line if research was skipped)*
   > **Output file:** [problem_dir]/round_1/ideas/idea_N.md
   > **Previously proposed ideas (avoid these areas):**
   > - [One-sentence summary from worker 1]
   > - [One-sentence summary from worker 2]
   > - ...

   Track: a list of surviving idea numbers (initially [1, 2, ..., K]) and the one-sentence summaries.

7. **Criticism and Revision Rounds.** For each round N from 1 to K-1, perform steps 7a, 7b, and 7c:

   **7a. Criticism.**

   Run `mkdir -p [problem_dir]/round_N/criticism` via the Bash tool.

   Invoke one `gan:discriminator-worker` agent per surviving idea, **all in parallel**. Each worker receives:
   > Critically evaluate the idea in the file below. Score it on ONLY the specified dimensions and provide detailed criticism.
   >
   > **Scoring dimensions:** [the scoring dimensions list, passed through exactly as received from the user]
   > **Idea file:** [problem_dir]/round_N/ideas/idea_X.md
   > **Output file:** [problem_dir]/round_N/criticism/idea_X_criticism.md

   **7b. Verdict.**

   Invoke a single `gan:discriminator-judge` agent, passing the list of criticism file paths:
   > Read the following criticism files and determine which idea should be evicted (lowest aggregate score).
   >
   > **Criticism files:**
   > - [problem_dir]/round_N/criticism/idea_X_criticism.md
   > - [problem_dir]/round_N/criticism/idea_Y_criticism.md
   > - ...

   The judge returns the eviction decision as text. Remove the evicted idea number from the surviving ideas list. Always execute the judge's decision — do not override it.

   **7c. Revision.**

   Run `mkdir -p [problem_dir]/round_[N+1]/ideas` via the Bash tool.

   Invoke one `gan:generator-revision-worker` agent per surviving idea, **all in parallel**. Each worker receives:
   > Revise and strengthen the idea based on the criticism provided.
   >
   > **Idea file:** [problem_dir]/round_N/ideas/idea_X.md
   > **Criticism file:** [problem_dir]/round_N/criticism/idea_X_criticism.md
   > **Output file:** [problem_dir]/round_[N+1]/ideas/idea_X.md

   **CRITICAL:** Each worker receives ONLY its own idea path and its own criticism path. Do NOT pass paths to other ideas or other criticisms.

8. **Final Criticism.** After the last revision round (round K), one idea remains. Run a final criticism round on it so the summary has a final score and any remaining weaknesses are documented.

   Run `mkdir -p [problem_dir]/round_K/criticism` via the Bash tool.

   Invoke a single `gan:discriminator-worker` agent:
   > Critically evaluate the idea in the file below. Score it on ONLY the specified dimensions and provide detailed criticism.
   >
   > **Scoring dimensions:** [the scoring dimensions list, passed through exactly as received from the user]
   > **Idea file:** [problem_dir]/round_K/ideas/idea_X.md
   > **Output file:** [problem_dir]/round_K/criticism/idea_X_criticism.md

9. **Summary.**

   Invoke a single `gan:summary-writer` agent:
   > Read all files in the problem directory and write a comprehensive summary.
   >
   > **Problem directory:** [problem_dir]
   > **Problem statement:** "[the user's problem, verbatim]"
   > **Scoring dimensions:** [the scoring dimensions list]
   > **Number of ideas:** [K]

   The summary-writer reads all files, writes `[problem_dir]/summary.md`, and returns a short summary. Display the returned short summary to the user.

## Important Rules

- **Use the Agent tool to delegate all creative and evaluative work.** You must NEVER generate ideas, revise ideas, score ideas, or write criticism yourself.
- **Do NOT read agent definition files.** Never read `.md` files from the `agents/` directory.
- **Do NOT read or write problem files yourself.** Workers handle all file I/O. You only create directories and pass file paths.
- You are a **neutral process manager**. Do not evaluate, influence, or validate ideas.
- Always execute the judge's eviction decision — do not override it.
- Ideation workers must be invoked **sequentially** (each needs to know about prior ideas via summaries).
- Discriminator workers and revision workers must be invoked **in parallel**.
- Preserve original idea numbering so ideas can be tracked through rounds (e.g. if idea 2 is evicted, round 2 contains idea_1.md and idea_3.md — not renumbered).
