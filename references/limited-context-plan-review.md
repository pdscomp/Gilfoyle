# Limited-context review of plan implementation

- When Paul asks for limited-context Gilfoyle, pass only repo/plan handles, exact base/final parent and changed-submodule SHAs, test environment and scope/side-effect constraints. Do not seed the independent reviewer with prior CLEAN verdicts, earlier reviewer explanations or inherited test totals.
- Review the full implementation range, not just the last docs commit. Read the plan as a fallible requirements source; implementing its example code faithfully does not prove its safety contract.
- Run scoped existing tests once where feasible, then probe missing cross-layer states. Passing tests cannot validate an engagement state the fixtures never populated.
- Independently reproduce defensible new findings before accepting them. Classify introduced, broadened and pre-existing behavior separately.
- If the user requested review and plan updates, edit only the existing plan in the project. Add an authoritative top-of-plan disposition, exact corrective files/guards, red/green acceptance, current SHAs and separate software/publication/hardware gates. Supersede stale instructions explicitly; do not implement fixes under planning authorization.
- Exercise executable plan test snippets in an isolated interpreter against unchanged code if useful. Label this as a snippet/probe check, not a pytest run or green implementation proof.
