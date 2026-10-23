# Feature branch scope mismatch

When asked to review a feature-named branch (e.g., `cc2-dsi`), verify the diff actually contains the claimed feature before reviewing it.

1. Check `git diff <merge-base>..HEAD`. If it is empty, the branch head may be the baseline or the work may be squashed. Use `git diff HEAD~1..HEAD` as the fallback scope.
2. Search the diff for the expected driver, binding, or algorithm name. If it is missing, report the mismatch as the first finding, not as a hidden assumption.
3. Review the actual changed code, not the branch name or ticket description.

This prevents a review of a feature that was never committed.
