# PLAN.md — Challenge-repo bug sweep (single ordered pass)

> Fixes **all 15 review findings + the intentional TICKET-002 invariant** on **one branch**
> (`fix/challenge-bug-sweep`), one PR titled `[TICKET-002] Fix notification invariant + repo bug sweep`.
> No PR split, no batching — findings are fixed **one at a time, in order (#1 → #15)**, each fully tested
> before the next. Finding numbers match the review report.

## Working discipline — applied to EVERY finding below

For each finding, in order:

1. **Apply the fix.**
2. **Add + run its named test** (listed with the finding) — red before the fix, green after.
3. **Run the existing suite** — every previously-green test (`HealthCheckTest`, `SequenceModelTest`, and every
   test added earlier in this pass) still passes; nothing regresses.
4. **Run the full suite for that surface** — `php artisan config:clear && php artisan test --parallel` for the
   app; the workflow smoke test for CI (see **Full tests**). If no full suite exists for a surface, it is
   **added** as part of this pass (the CI full-workflow test does not exist yet → added).

Only when steps 2–4 are green do we move to the next finding. One finding = one small, reviewable diff slice.

## Step 0 — two foundations laid first (so each later fix is small and testable)

These are enablers built up front; each also *is* the fix for a specific finding, noted inline.

- **CI script extraction (fixes #10).** Move the review logic out of the YAML heredocs into a committed,
  unit-testable `.github/scripts/review.py`, and delete the dead bash block (lines 50–64,
  `CLAUDE_MD`/`PR_DIFF`/`CHANGED_FILES`/`TEST_OUTPUT`/`$TICKET` — computed, never used). This makes
  #1/#3/#4/#5/#9/#11/#12/#15 assertable in `tests/ci/test_review.py` instead of only via a live PR.
- **Notification-invariant infra (basis for TICKET-002, #7, #14).** `sequence_notification_outbox` table +
  driver-aware DB triggers (`AFTER INSERT`, `AFTER UPDATE OF status`, `WHEN NEW.status NOT IN <terminal>`,
  generated in PHP from `Sequence::TERMINAL_STATUSES` so SQL keeps one source of truth) + a
  `SequenceNotificationPolicy` (`canNotify()` / `shouldDispatchOnUpdate()`) + a
  `DispatchPendingSequenceNotifications` drain command (transactionally claims rows, stamps `dispatched_at`).
  The trigger is the **sole** outbox writer (no double-dispatch); the observer and repository only drain.

---

## Findings — fixed and tested in order

### #1 — Ticket misclassification (CI)
- **Fix:** detect the ticket from the PR **title** (`[TICKET-ID]` is mandated by CLAUDE.md) + branch name;
  demote file-keyword matching to a most-specific-first fallback (`userpaymentobserver`/`sendpayment` → 003
  **before** `sequenceobserver` → 002) and **include TICKET-004**.
- **Named test:** `test_review.py::test_ticket_from_pr_title`, `::test_ticket003_not_misclassified_as_002`
  (fixture: real changed-file list touching `UserPaymentObserver.php`).

### #2 — `--parallel` mandated but `brianium/paratest` absent (app/tooling)
- **Fix:** add `brianium/paratest` to `require-dev`; commit `composer.lock` for reproducible resolution.
- **Named test:** `ParatestAvailableTest::test_paratest_is_installed` (asserts `class_exists(\ParaTest\...)`);
  the full `php artisan test --parallel` run is itself the integration proof.

### #3 — API/curl error crashes the review step → no comment posted (CI)
- **Fix:** `curl --fail-with-body -sS -w '%{http_code}'` + one retry on 429/5xx; `try/except` around
  `json.loads` and `content[0].text`; on any failure write a fallback comment (**"Automated scoring pending —
  routed to a human reviewer"**, no negative wording per CLAUDE.md) and set the manual-review sentinel. Add
  `continue-on-error: true` to the step **and** `if: always()` to "Post review comment" so a candidate always
  gets a comment.
- **Named test:** `test_review.py::test_api_error_writes_manual_review_comment` (mocked non-200 / empty stdout).

### #4 — Score-parse miss → 0 → auto-decline (CI)
- **Fix:** broaden the regex to tolerate markdown/bold and phrasing
  (`Weighted Score:\**\s*(\d+(?:\.\d+)?)` + a "score of X" alternate); when unparseable, route to
  🟡 manual-review via a sentinel — **never** 0. Auto-decline fires only on a *parsed* score `< 5`.
- **Named test:** `test_review.py::test_score_parse_bold_plain_reworded_absent` (absent → manual-review, not 0).

### #5 — Unmatched ticket → blank rubric (CI)
- **Fix:** add TICKET-004; when still undetermined, inject a generic note listing all ticket titles **and**
  force the header to 🟡 manual-review — never auto-score against an empty rubric.
- **Named test:** `test_review.py::test_unknown_ticket_forces_manual_review`.

### #6 — `tee` masks the test exit code, and it's unused (CI)
- **Fix:** `set -o pipefail` (or `${PIPESTATUS[0]}`) so `test_exit_code` reflects PHPUnit; then **consume** it —
  pass `tests_passed` + `composer_failed` into the prompt so the 25% Tests dimension is scored on ground truth.
- **Named test:** `tests/ci/pipefail.bats::test_failing_phpunit_sets_tests_passed_false` (dummy failing test →
  step reports `tests_passed=false`).

### #7 — `findOrFail` throws on a cascade-deleted sequence (app)
- **Fix:** in `NotifySequenceUpdate::handle()`, replace `findOrFail($id)` with `find($id)` + skip-log early
  return, and set `public bool $deleteWhenMissingModels = true`. (Also add the terminal `canNotify()`
  safety-net here — see TICKET-002.)
- **Named test:** `NotifySequenceUpdateJobTest::test_deleted_sequence_is_skipped_not_thrown`.

### #8 — `phpunit.xml` sets the ignored legacy `CACHE_DRIVER` (app/config)
- **Fix:** rename the env to `CACHE_STORE` (what Laravel 12's `config/cache.php` reads; matches `.env.example`).
- **Named test:** `CacheConfigTest::test_array_store_is_pinned` (`config('cache.default') === 'array'`).

### #9 — Fork PRs can't be reviewed (CI)
- **Fix:** do **not** switch to `pull_request_target` (it would hand secrets to untrusted candidate code —
  exfiltration risk). Guard the job: if `github.event.pull_request.head.repo.fork == true`, skip the API/test
  steps and post a clear manual-review comment.
- **Named test:** `test_review.py::test_fork_pr_skips_api_and_posts_manual_review`.

### #10 — Dead bash block / logic buried in YAML (CI)
- **Fix:** delivered by **Step 0** (logic extracted to `review.py`; dead bash removed).
- **Named test:** `tests/ci/test_workflow_structure.py::test_review_logic_is_external_and_no_dead_bash`
  (asserts the step calls `review.py`, and the workflow contains no `python3 <<` heredoc and none of the
  unused context vars).

### #11 — Step outputs set but never consumed (CI)
- **Fix:** drop `diff_size`; `composer_failed` + `test_exit_code` are now *consumed* by #6.
- **Named test:** `tests/ci/test_workflow_structure.py::test_no_orphan_github_outputs`
  (every `>> $GITHUB_OUTPUT` key is referenced later in the workflow).

### #12 — Silent truncation of diff / test output (CI)
- **Fix:** append `[...truncated N of M chars...]` when cutting `diff`/`test_out`/`claude_md`; upload the full
  diff as a workflow artifact for the human reviewer.
- **Named test:** `test_review.py::test_truncation_marker_emitted` (input > limit → marker present in prompt).

### #13 — `Company::scopeSearch` doesn't escape LIKE wildcards (app)
- **Fix:** `addcslashes($term, '%_\\')` + `like ... ESCAPE '\\'`.
- **Named test:** `CompanySearchScopeTest::test_wildcards_are_treated_literally` (`q=%`, `q=a_b` match only
  literal rows, not everything).

### #14 — Observers fire only on `updated()`; mass writes bypass (app)
- **Fix:** delivered by the **Step 0** outbox + DB triggers — coverage is independent of the write path
  (`query()->update()` / `insert()` still record + dispatch).
- **Named test:** `SequenceNotificationFlowTest::test_bulk_update_records_and_dispatches`,
  `::test_terminal_bulk_update_notifies_nobody` (`Bus::fake()` + `RefreshDatabase`).

### #15 — Threshold label off-by-one (CI)
- **Fix:** relabel the middle band to "Score 5–7.9/10" so it matches the actual `[5, 8)` range.
- **Named test:** `test_review.py::test_score_band_labels` (5.0 & 7.9 → yellow "5–7.9", 8.0 → green, 4.9 → decline).

### Intentional TICKET-002 invariant — "terminal sequences never receive notifications" (app)
- **Fix:** `SequenceObserver` becomes thin (on `saved`, kicks the drain — no direct dispatch); the outbox
  trigger's `WHEN NEW.status NOT IN <terminal>` enforces the invariant below Eloquent; `handle()` keeps the
  final `canNotify()` skip-log safety-net for a row whose sequence went terminal after enqueue.
- **Named test:** `SequenceNotificationFlowTest::test_terminal_sequence_never_notifies`,
  `::test_active_sequence_still_notifies`, `::test_job_safety_net_skips_stale_terminal_row`.

### Bonus hardenings (bundled, each tested)
- **Money precision:** `'amount' => 'decimal:2'` on `Sequence` + `UserPayment` — `MoneyCastTest::test_amount_is_decimal_2` on both.
- **Repo hygiene:** `.gitignore` ignores `database/*.sqlite` — verified by `git diff main...HEAD` showing no
  `.sqlite` leak (tests use `:memory:`, unaffected).

---

## Endpoint tests — every route gets a Feature test

| Endpoint | Test | Cases |
|---|---|---|
| `GET /api/v1/health` (exists) | `HealthCheckTest` (kept green) | 200 + `{status: ok}` |
| `GET /api/v1/sequences` (reference controller added — gives TICKET-001 a real "existing pattern" to follow) | `Api/V1/SequenceIndexTest` | happy path, **422** validation, pagination, `status` filter, **empty result** |
| `GET /api/v1/companies/search` | owned by **TICKET-001** (out of bug-sweep scope). If implemented, `Api/V1/CompanySearchTest` with the same matrix (happy / 422 / empty / pagination / status). Meanwhile #13's `CompanySearchScopeTest` covers the query layer. |

## Full tests (one per surface — add if missing)

- **App full suite — exists.** `php artisan config:clear && php artisan test --parallel` runs both the `Unit`
  and `Feature` suites (`phpunit.xml`). This is the regression gate rerun after **every** finding above.
- **CI full-workflow test — does not exist → added.**
  - `tests/ci/test_review_end_to_end.py` drives `review.py` through the whole path on a synthetic
    `pull_request` event (detect ticket → build prompt → **mocked** Anthropic call → parse score → write
    comment → set `score` output), asserting: happy path posts a scored comment, and an injected API error /
    unparseable score both post a 🟡 manual-review comment (the fail-open invariant, proven, not hoped).
  - `actionlint` + `shellcheck` run on `review-candidate.yml` in CI to catch the pipe/quoting class of bug
    (#6) at lint time.

---

## Open questions

1. **`notify_on: any_update` in the DB trigger, or app-layer only?** Triggers fire on *status change*;
   non-status edits notifying is a repository/observer behaviour. Default: `status_change` fully covered
   end-to-end; `any_update` app-layer only (SQL enforcement needs an unconditional `AFTER UPDATE` + edit-spam
   dedupe — materially larger).
2. **Which prod DB dialect (mysql vs pgsql) for the trigger variants?** Only the sqlite variant is exercised
   by CI (`:memory:`). Default: ship sqlite + mysql + pgsql via driver match, treat prod as mysql; if
   confirmed, drop the unused variant and add a migration smoke-test on the real dialect.
3. **The candidate-facing template was repurposed for this plan.** This document now lives at repo-root
   `PLAN.md` (moved from `challenge/PLAN.template.md`), matching the README deliverable. Note
   `challenge/PROBLEM.md:94` still links candidates to a template path and would need updating if the
   original template must be restored.
4. **CI workflow is grader infrastructure.** Per instruction it ships in this one branch alongside the app
   fixes; flag in the PR body that `review-candidate.yml` is hiring-team-owned so reviewers expect it in the diff.
