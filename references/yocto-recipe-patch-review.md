# Reviewing patches embedded in Yocto/OE recipes

Common in OpenCentauri cosmos: the PR diff is not applied source — it is a
`*.patch` file under `meta-*/recipes-*/.../files/` plus a `PR`/`SRCREV` bump
in the `.inc`/`.bb`. Reviewing the patch text alone misses context; reviewing
the working tree shows nothing.

## Recipe

1. Get the pin: `SRCREV` and branch in the recipe `.inc` (e.g.
   `meta-opencentauri/recipes-apps/klipper/kalico_2026.02.00.inc`).
2. Fetch the exact upstream file the patch touches at that SRCREV:
   `https://raw.githubusercontent.com/<org>/<repo>/<SRCREV>/<path>` via
   web_extract. Do NOT review against upstream `main` — it may have moved.
3. Mentally (or with `git apply` on a scratch checkout) apply the patch and
   read the resulting function end-to-end, not just the hunks. Hunk context
   hides what the removed lines did.
4. Check the delivery mechanism: for MCU firmware, a `PR = "rN"` bump (or
   equivalent) is often the ONLY thing that makes the target device actually
   flash the rebuilt firmware. A patch without the bump is a no-op on
   hardware — flag it if missing; credit it if present.
5. No test suite is normal (per SKILL.md). Build success claims in the PR
   body ("N bitbake tasks succeeded") plus on-device validation notes are
   the evidence; quote them in the verdict.

## Pitfall: branch-rewrite side-effect audit

When a patch REWRITES a conditional block (new state machine, refactored
if/else), enumerate every branch of the OLD code and list its side effects
(state writes, forward/report calls, counter updates). Then check each new
branch preserves them. Real example from cosmos PR #262: the rewrite dropped
`load_cell_probe_report_sample()` from the `!have_last_*` first-sample
branch — original forwarded the first sample unconditionally, new code only
seeded state. Benign there (consumer guards on `is_homing`), but the audit
is what catches it when it isn't benign.

## Pitfall: "two gists/docs" scavenger hunt

When the user references external evidence ("see the gists", "the analysis
doc"), find them in this order before asking: PR body → PR comments
(`gh pr view N --comments`) → linked issue body + comments → user's gist
index (`https://gist.github.com/<user>`). Docs with the same name may exist
in both the gist AND the PR at different lengths — diff them and say which
one you used.
