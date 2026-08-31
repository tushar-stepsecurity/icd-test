# icd-test — imposter commit detection regression suite

Fixtures for verifying the imposter-commit analysis fix
(`fix/imposter-commit-branch-walk-false-positives`). One workflow per scenario.

## Prerequisites

1. **Onboard this repo to StepSecurity `int`.** Nothing here produces a detection
   until the run is correlated by the environment under test.
2. **A scenario only exercises the analysis on a cache miss.** The `ImposterCommits`
   row is keyed on `action` + `sha` and shared globally, so the *first* run of a
   scenario is the meaningful one. To re-test, delete the row first:

   ```
   aws dynamodb delete-item --table-name ImposterCommits \
     --key '{"action":{"S":"<action>"},"sha":{"S":"<sha>"}}'
   ```

   Without that, every later run is a 2ms cache hit and proves nothing about the
   analysis.

## Fixture repos

| repo | what it provides |
|---|---|
| `tushar-stepsecurity/icd-test-action` | main, `releases/v1`, a lightweight and an annotated tag, and two orphaned commits whose branches were deleted |
| `tushar-stepsecurity/icd-test-bigrepo` | 521 branches, plus an orphaned commit — exceeds the 500-branch cap |

## Scenarios

| # | Scenario | Action@sha | Expected detection | Evidence to collect |
|---|---|---|---|---|
| s01 | SHA is the head of a non-default release branch | `icd-test-action@44164e34` | `Action-Uses-Commit-From-Non-Default-Branch` | `commit is the head of a branch, skipping branch walk` with `branches=[releases/v1]`. **No** `branches?per_page=100` call. |
| s02 | SHA is contained in a release branch but is not its head | `icd-test-action@76f03925` | `Action-Uses-Commit-From-Non-Default-Branch` | `commit is contained in branch` (Debug) with `branch=releases/v1` and a low `compared` count — proves prioritisation ran |
| s03 | SHA is the head of the default branch | `icd-test-action@bd121c99` | **none** | no `[ImposterCommits]` detection line for this action |
| s04 | SHA is contained in the default branch | `icd-test-action@0488f6d3` | **none** | `compare(main...sha)` returns `behind`; no detection |
| s05 | Genuine imposter — commit shares history, branch deleted | `icd-test-action@f3f99fa4` | **`Action-Uses-Imposter-Commit`** | `Imposter commit detected` with `verdict=` empty, then `sent imposter commit review request`. Proves real detection still works. |
| s06 | Genuine imposter — unrelated history, so compare returns `404 No common ancestor` | `icd-test-action@f84cebb0` | **`Action-Uses-Imposter-Commit`** | as s05. Proves the one 404 that *is* an answer is still honoured. |
| s07 | Reference is an annotated tag, so the runner logs the tag object SHA | `icd-test-action@v1-annotated` | `Action-Uses-Commit-From-Non-Default-Branch` | `resolved annotated tag chain to commit SHA` (Debug); row keyed on the resolved commit `44164e34`, not the tag object `d5084497` |
| s08 | Absence cannot be proven — 521 branches, commit genuinely on none | `icd-test-bigrepo@44f5d443` | `Action-Uses-Commit-From-Non-Default-Branch` (**downgraded**) | `not every branch could be compared, absence cannot be proven` with `branches=521`, `listed_all=true`. **Must not** be an imposter verdict. |
| s09 | The incident: codeql-action v3, 438 branches, sha is the `releases/v3` head | `github/codeql-action@6f5948df` | `Action-Uses-Commit-From-Non-Default-Branch` | `commit is the head of a branch` with `branches=[releases/v3]`; analysis completes in under a second, not four minutes |
| s10 | Widely used sha contained in the default branch | `actions/checkout@df4cb1c0` | **none** | the Aug 17 false-positive sha — must stay clean |
| s11 | Widely used sha that is a `releases/v4` head | `actions/checkout@11d5960a` | `Action-Uses-Commit-From-Non-Default-Branch` | fast path hit |
| s12 | Action from this repo, referenced by full name | `icd-test/.github/actions/local-fixture@main` | **none** | `same-repo action, skipping analysis`; entry carries `icd_suppressed=true` |

## Scenarios that need a manual step

| # | Scenario | How to run | Expected |
|---|---|---|---|
| s13 | Suppression read path | Set `suppress_icd=true` on the `GitHubActionDetails` row `tushar-stepsecurity/icd-test-action`, then re-run s05 with its row deleted | `action is suppressed, skipping analysis`; no detection |
| s14 | Suppression write path | Call the suppress-ICD API for `tushar-stepsecurity/icd-test-action` | succeeds; previously always returned `ValidationException` |
| s15 | Verdict demotion | Set `verdict=false_positive` on the s05 row, re-run s05 | detection recorded but `is_suppressed=true`, `suppressed_by=FalsePositiveReview` |
| s16 | Verdict pending | Set `verdict=pending`, re-run s05 | detection alerts as normal; `review already requested … skipping review request` |
| s17 | Branch-head endpoint unavailable | Not reproducible on github.com — verify on GHES, where the `groot-preview` media type may be rejected | `branch head check failed, falling back to the branch walk` with `status=415`; verdict still correct via the walk |

## Fleet-wide regression: the log line that must be gone

```
fields @timestamp | filter @message like /Query key condition not supported/ | stats count() by bin(5m)
```

That fired once per action, per run, per tenant before the fix. It should now be zero.
