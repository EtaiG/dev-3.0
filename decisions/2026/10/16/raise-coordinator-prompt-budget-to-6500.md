# Raise the coordinator prompt budget from 6 210 to 6 500

## Context

`COORDINATOR_PROMPT` was capped at 6 210 characters (`preset-prompt.test.ts`), the length it
had before the events block, so a coordinator task never lost Windows launch room. A paired
guard in `agent-command-line-budget.test.ts` required 26 536 (gemini) / 26 525 (custom agent)
characters of brief room after the preamble. At 6 187 characters a ~200-character rule (Seq
2159) no longer fit, and the two guards disagreed: the gemini floor failed at ~6 200 raw.

## Investigation

Audit Seq 2160, on main 0b1952ae0. Only the dev3 protocol body travels as a file, and only
for Claude and omp. Every adapter still interpolates the task's first prompt, preamble
included, into argv, and on Windows that reaches `CreateProcess` (32 767). Measured with
the real `resolveAgentCommand` on win32: a prompt padded to exactly 6 500 leaves 26 239
(gemini) / 26 228 (custom) characters for the brief. That is far beyond any realistic task.
Codex, Cursor and OpenCode inline a ~27.7K protocol, so their coordinator tasks were already
past the ceiling at 6 187. No value of this cap changes that.

## Decision

The cap becomes 6 500 by the user's call. The floors move with it, to 26 200 / 26 190: about 38
below the measured room, so a prompt at the cap with a few more escaped quotes still passes
both guards instead of only one. The cap is a budget for brief room, not an OS limit, and
the test comments now say so.

## Risks

Windows coordinator tasks on gemini, Copilot, Claude, omp and custom agents give up ~300
characters of brief room. Nothing here was run on a real Windows machine: how real agent
CLIs resolve npm shims was not checked, and a `.cmd` shim run by `cmd.exe` caps the line at
8 191 regardless.

## Alternatives considered

Trimming other prose to stay under 6 210 (Seq 2159's first plan): it works, but it spends
wording on a number that protects no real launch. Deriving the floor from the cap in code:
cleaner, but out of scope for a number change.
