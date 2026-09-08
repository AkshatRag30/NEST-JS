# How to Use These Prompts

Each file in this folder is one complete, self contained prompt you can hand to Claude Code to build one phase of LearnBridge. They exist so that the plan in the parent `planning` folder turns into actual working code without you having to translate a PRD into an instruction yourself every time.

## The honest tradeoff, said once

The whole reason the planning folder ends with a checklist item telling you to build one feature completely alone, with no notes and no AI, open is that reading generated code and writing code yourself build different muscles. Using these prompts for every phase will get you a finished, working LearnBridge fast, but it will not, by itself, make you the backend engineer the plan is aiming for. The version of this that actually works is to read every diff Claude Code produces before accepting it, ask it to explain any line you do not immediately understand, and seriously consider building phases 1 through 3 yourself by hand first, using the PRDs alone, and only start leaning on these prompts once the pattern already feels familiar. How you use them from here is your call, this is just being direct about what each choice actually costs.

## How each prompt is built

Every prompt tells Claude Code exactly which planning files to read before writing anything, so the ground truth always lives in one place, the PRDs, and these prompts never drift from it. Every prompt restates that phase's explicit trap to avoid, since that is the single most valuable sentence in each PRD. Every prompt asks for a small amount of targeted testing during the phase itself, focused on the riskiest logic, with full test coverage deliberately saved for the dedicated testing phase.

## Order of use

Run them in order, phase 1 through phase 10, in a fresh or ongoing Claude Code session with `LearnBridge-Capstone` open as the working directory. Do not skip ahead, phase 5 deliberately changes something phase 4 built, and phase 7 assumes phases 1 through 6 already exist and pass their own checks. After each prompt finishes, actually run the app and manually verify that phase's acceptance criteria from its PRD before moving to the next one, do not just trust that the code compiles.

1. [phase-1-prompt.md](phase-1-prompt.md)
2. [phase-2-prompt.md](phase-2-prompt.md)
3. [phase-3-prompt.md](phase-3-prompt.md)
4. [phase-4-prompt.md](phase-4-prompt.md)
5. [phase-5-prompt.md](phase-5-prompt.md)
6. [phase-6-prompt.md](phase-6-prompt.md)
7. [phase-7-prompt.md](phase-7-prompt.md)
8. [phase-8-prompt.md](phase-8-prompt.md)
9. [phase-9-prompt.md](phase-9-prompt.md)
10. [phase-10-prompt.md](phase-10-prompt.md), optional, and built as a separate throwaway project, not inside LearnBridge itself

## What to do if Claude Code's output disagrees with a PRD

Trust the PRD first, then think about whether the PRD or the code is actually right before changing either one. A few of these prompts flag a real, deliberate design decision left open on purpose, for example the exact transaction outcome for a failed payment in phase 5, worth reading closely rather than skimming.
