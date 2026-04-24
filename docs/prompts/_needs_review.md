# Prompts needing review

These prompts could not be confidently matched to a merged PR. Options for each:
- Match to a PR manually -> move to `_archive/` with provenance header
- Recognize as abandoned -> delete
- Recognize as still-relevant work -> convert into a backlog story under `docs/product/backlog/<epic>/`

Current state: one prompt is still active because it appears tied to open frontend work. Previously listed abandoned prompts have been pruned.

| Filename | Reason not matched | Proposed action |
|----------|--------------------|-----------------|
| `fe-google-calendar-test-coverage.md` | No confidently matched merged PR. Closest related work is open FE PR `joe-bor/FamilyHub#128`, so this is not archive-ready yet. | Keep in `docs/prompts/`; if FE #128 merges and clearly covers this scope, archive later; otherwise convert residual work into a backlog story. |
