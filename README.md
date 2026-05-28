# stern-code-review

A Claude Code skill that performs a blunt, senior-level code review focused on
production risk — correctness, data safety, security, operational reliability —
not style nits.

## Purpose

Use it when you want a rigorous reviewer who assumes the code will run in
production, treats clever code as guilty until proven useful, and treats missing
tests as unprotected behavior. It is *not* for brainstorming, greenfield
implementation, or purely stylistic feedback.

## Usage

```
/stern-code-review <files, a diff, or a description>
```

Or just ask for "a stern code review" of something in context. Arguments are
treated as the review target; if scope is unclear the skill asks before
reviewing everything, and for large changesets it starts with the riskiest files.

## How it works

The skill steers the review with a few rules:

- **Ranked priorities** — correctness → data safety → security → reliability →
  simplicity → testability → maintainability → performance → style. Formatting
  nits are ignored unless they hide a deeper problem.
- **Verify before filing** — a finding must be confirmed in the code shown, or
  marked `suspected` with the assumption stated. Empty sections are a valid
  result; the skill does not invent issues to fill them.
- **Mechanical verdict** — the verdict is derived from the highest-severity
  finding, not chosen by feel:

  | Highest finding              | Verdict             |
  | ---------------------------- | ------------------- |
  | any `blocker`                | request changes     |
  | design structurally unsafe   | reject              |
  | only `major`                 | approve with fixes  |
  | only `minor`, or none        | approve             |

- **Conditional output** — only the verdict and a one-line closing sentence are
  mandatory. Other sections (Findings, Suspicious choices, Tests I expect,
  Minimal acceptable fix, Optional cleaner version) appear only when they have
  content, so a trivial clean change yields a two-line review.

Each finding carries a severity (`blocker` / `major` / `minor`), a confidence
(`confirmed` / `suspected`), a location, the problem, why it matters, and a
suggested fix.
