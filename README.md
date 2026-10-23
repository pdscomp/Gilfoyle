# Gilfoyle — Hermes edition

> "Your code is bad and you should feel bad—but I'll explain exactly why."

A fork of [miqcie/gilfoyle](https://github.com/miqcie/gilfoyle) tuned for the
[Hermes Agent](https://github.com/NousResearch/hermes-agent) skill system.
Upstream reviews a change and returns severity-ranked, evidence-backed
findings with a verdict. That contract is unchanged here. What follows is what
a few hundred real reviews added on top of it: how to pick a scope when the
target is ambiguous, how to judge a port or a drop-in replacement against its
reference, what to do when there is no test suite, and how to run the review in
a fresh context without trusting whatever comes back.

Original skill and review contract by **Chris McConnell
([@miqcie](https://github.com/miqcie))**, MIT licensed. See [Credit](#credit).

## What we added

- **Scope discovery table.** Five rungs, first match wins: named files, then
  parity/port framing, then the working tree, then the last commit, then ask.
  Reviewing the wrong scope is the most expensive mistake available, and a
  reviewer that silently expands to the whole repo is worse than useless.
- **Whole-project claims.** When the target replaces, ports, or claims parity
  with something, the review is the branch diff against the merge base, and
  findings quote `file:line` from *both* sides. "The math looks the same" is
  not a finding.
- **Severity tiers with definitions.** Critical = breaks a documented contract
  (silent wrong answers, data loss, unrecoverable state). Major = divergence
  under realistic conditions. Minor = hardening and cleanup. Tiers get a
  consequence, not a feeling.
- **A dispatchable persona block.** The system prompt is part of the skill, so
  the review can be handed to a subagent with the same voice and the same
  standards. Fresh context is the point: the context that wrote the code is
  the worst place to judge it.
- **Run the tests once, say so.** "I read it carefully" is not a test result.
  The report carries the command and the count, or states plainly that no
  suite exists.
- **Pitfalls from boards and Yocto.** No test suite is normal for bring-up, so
  review the diff, the bindings, and the schematic instead. Yocto recipe
  patches are not applied source: read the post-patch file at the recipe's
  `SRCREV`. Feature-named branches lie about their contents; check the merge
  base before deciding what you are reviewing. Never reach for a vision model
  to read a diff.
- **Delegated reviews treat findings as leads.** A subagent's output is
  verified locally before anything is fixed: expand the globs, resolve the
  symlinks, re-run the build. A finding that survives thirty seconds of
  checking is usually real; one that doesn't was invented to sound thorough.
- **`references/` playbooks** for the reviews that kept needing a checklist:
  embedded DSI bring-up, parity against a merged reference driver, Yocto
  recipe patches, the feature-branch scope trap, and limited-context plan
  review.

## What this fork does not have

Upstream also ships a `gilfoyle` CLI, a JSON review schema, a fixture-based
eval suite, and generated wrappers for Claude Code, Codex, and Cursor. None of
that is here. This repo is one canonical `SKILL.md` plus reference playbooks,
for a harness that loads skills as markdown. If you want the CLI or the
evaluations, use [upstream](https://github.com/miqcie/gilfoyle).

## Install

Hermes indexes any directory under its skills tree that contains a
`SKILL.md`:

```bash
git clone https://github.com/pdscomp/Gilfoyle.git ~/.hermes/skills/external/gilfoyle
```

Then invoke it by name from a session, or with a `/gilfoyle` command if your
setup maps slash commands to skills.

## Use

```text
/gilfoyle                                  # working-tree diff, else last commit
/gilfoyle src/auth.py                      # named file and its call sites
/gilfoyle the JWT refresh token flow       # topic lens over the current change
/gilfoyle <repo> as a replacement for X     # whole branch vs the reference
```

Consequential reviews belong in a subagent or a new session fed the diff.
Blocking findings get reproduced before anyone changes code.

## Review contract

Every review opens with one line of scope and closes with a verdict.

Each finding carries: severity, `file:line`, what is wrong, why it matters,
and the smallest fix — with a snippet when the snippet is shorter than the
prose. Findings are ordered Critical, Major, Minor and empty tiers are
skipped. Style preferences and low-confidence trivia are suppressed; silence
beats manufactured competence.

The verdict is `ship`, `fix-then-ship`, or `back-to-the-drawing-board`. Clean
changes get a clean verdict. Humor is welcome and must sit on top of a real
finding — a joke with nothing under it is noise, and he despises noise.

## References

| File | Use |
|---|---|
| `references/embedded-dsi-review-notes.md` | U-Boot/kernel DSI panel bring-up checklist when no tests exist |
| `references/parity-merged-reference-drivers.md` | Driver that merges logic from several upstream references |
| `references/yocto-recipe-patch-review.md` | `*.patch` under a Yocto recipe plus a `PR` bump |
| `references/feature-branch-scope-trap.md` | Branch name promises code the diff does not contain |
| `references/limited-context-plan-review.md` | Reviewing a plan when you cannot load the whole system |

## Credit

Ported from and extended past **[miqcie/gilfoyle](https://github.com/miqcie/gilfoyle)**
by **Chris McConnell** (`mcconnell.chris@gmail.com`), MIT licensed.

The persona prompt, the six review lenses, the severity tiers, and the output
format come from `gilfoyle-tech-reviewer.md` at commit
**[`abed46f`](https://github.com/miqcie/gilfoyle/blob/abed46feeeafa088a6f90709c265cb1907ebb5fd/gilfoyle-tech-reviewer.md)**
(2026-06-18, "fix: drop 'comprehensive/multi-perspective' framing for
congruence (#2)"). That is the last revision of the file that resembles this
skill; the project has since rewritten the reviewer into a canonical
`SKILL.md` and removed `gilfoyle-tech-reviewer.md` from `main`, so the
permalink is the only place the overlap is visible. We cloned at `master`
HEAD `2b2d4e72` purely to keep provenance of the fork point; that snapshot
shares no text with this skill.

Everything else here is local: the scope-discovery ladder, whole-project and
parity review, the pitfall playbooks, verification gates, and the delegated
review rules.

## License

MIT. Original copyright © 2025 Chris McConnell; modifications © 2026 Paul
Swenson. See [LICENSE](LICENSE).
