# Evaluations, comparisons and optimizer handoffs

Read this before you execute test cases, compare or score prompts, test a skill's trigger, or hand a prompt to an optimizer. Step 8 of `SKILL.md` says when each applies.

## Before running anything

- Use what the host already has. An eval runner configured in the repository (promptfoo or similar), a skill-testing loop (Anthropic's skill-creator, `claude plugin eval`) or stored production traces beat a harness improvised in chat. Write the cases in that tool's format and run them there.
- Agree the cost first when it is more than a handful of runs: how many runs, on which model, and roughly what they cost in tokens or time.
- Tests have no consequential side effects. Stub or forbid sends, writes, payments and deletions, and keep secrets and personal data out of the cases.
- An assertion that depends on a model fact is only as good as the fact. Check it against the vendor's documentation before you count a pass or a failure on it, in your own tests and in results someone else reports.

## Isolated evaluation

1. Fix the cases and their assertions before the candidate is final. Where you can, have the held-back cases written by a person or a fresh context that has not seen the candidate, because an author's own tests reward the author's own habits.
2. Run the candidate and a baseline on the same cases with the same model, settings and checks, each run in a fresh context that sees only its own prompt and input. The baseline is the previous version for a revision, and no prompt or the user's original for a new one.
3. Let a script check whatever a script can check: exact strings, schema validity, the real parser, counts.
4. For the rest, use a separate fresh-context grader that cannot tell which output came from which prompt. It decides each assertion and quotes the passage that decides it. Tell it that length, tables and a confident tone are not evidence of quality. For a preference between two outputs, show both unlabelled and repeat with the order swapped, because judges favour a position; count a preference only when it survives the swap.
5. Repeat runs when affordable (three times), because one run misleads. Count timeouts and other infrastructure failures apart from wrong answers, and keep the raw replies.
6. Report counts: paired wins, losses and ties, and passes out of the total, with tokens and time where available. With few cases, give counts and not percentages, and treat a difference of one case as a tie. When both fail a case, score it a tie and say which failure remains. On a tie, keep the original.
7. Revise for at most three rounds on the cases not held back, then run the held-back cases once and report them as they fall. A held-back case that informs an edit is used up, so draw new ones before claiming that the result transfers.

When the runtime cannot isolate contexts, or the user or the host has ruled runs out, leave the cases `[PROPOSED]` and say how to run them.

## Compare and evaluate jobs

Fix the rubric before judging, weighted for what the user will use the prompts for. Score each prompt for the job it claims to do. Separate what was measured from what is your judgment, prefer blind fresh-context judging, and say so if one candidate is yours, since that score is the least reliable in the table. A rubric score is an opinion; only a head-to-head run on the user's real tasks decides.

## Skill triggers

For a skill, also test the description: realistic requests that should trigger it and near-misses that should not, each run several times. Tune the description on one part of the set and judge it on the part you held out.

## Optimizer handoff

Step 8c of `SKILL.md` says when to offer one. When the user asks for it, stop hand-tuning and specify, for a reflective optimizer such as GEPA (through DSPy or its own package) or a comparable tool:

- the task signature: the inputs, the outputs and their types
- train and validation sets, plus a test set the optimizer never sees
- a metric that returns a score and written feedback on why each failure happened, because reflective optimizers learn from that text and a bare number gives them nothing to reflect on
- a strong reflection model, and a small budget first to prove the metric before the full run
- the baseline score to beat, measured on the same test set

Selection happens on the validation set, and the test set is run once at the end. Check the installed library's current API before writing parameter names, because these libraries change quickly.
