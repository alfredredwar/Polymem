# Harness gate audit — 2026-10-01

Audit of Polymem's pre-publish safety checks: are they **wired into CI and proven to
fail**, or merely present as scripts nobody runs?

> **Status: superseded in part.** The first version of this audit was measured at
> `bdce4bb`. This revision re-measures at `2be9fc2`, where the forged-suite hole and the
> floor-slack hole have since been closed. Sections 3, 4 and 5 are kept as the record of
> what was measured then; the current state is in §0. Do not read a historical gap below
> as an open one — check §0 first.

All figures below were **measured at committed HEAD `bdce4bb`**, not quoted from an
earlier audit. Where an earlier audit's number is repeated it is labelled as stale.

This document replaces a citation to `10_BRAIN/runbooks/polymem-harness-gate-2026-10-01.md`,
a path that **does not exist** in this repository — there is no `10_BRAIN/` tree here, in
either local clone, in any branch, or in any stash. The findings below were re-derived from
scratch rather than recovered from that dead reference, which is why several of them now
differ from it. See "Corrections to the earlier record" below.

**Verdict at `bdce4bb`: `GATE: GAPS`.** Every guard was genuinely wired and every guard
fired. Two coverage gaps remained open (one HIGH), one previously-reported gap was already
fixed, and one previously-reported gap had **widened**.

---

## 0. Re-measurement at `2be9fc2` — 2026-10-04

The state above is historical. HEAD has moved five commits past the `bdce4bb` baseline, and
two of them closed the gaps this document reported. Re-run from a real clone, control first:

| Gap | Status at `2be9fc2` | Evidence |
|---|---|---|
| **GAP 1** — forged suite result line (HIGH) | **CLOSED** | Forgery now `npm test` **exit 1** (`reported pass=5 but emitted 0 passing assertion line(s)`) and `docs:check` **exit 1** |
| **GAP 2** — floor slack (MEDIUM) | **CLOSED** | `SUITE_COUNT_FLOOR` is now the real count (17) and `TOTAL_ASSERTION_FLOOR` 559 — **zero slack**; deleting 2 suites → **exit 1** |
| **GAP 2b** — orphaned floor row (MEDIUM) | **CLOSED** | Delete a suite *and* its floor row → **exit 1**; the inverse check now exists |
| **GAP 3** — authorship guard reads wrong repo | was already fixed | unchanged since `bdce4bb` |
| **No pre-commit hook** (MEDIUM) | **STILL OPEN** | `.git/hooks/` holds only samples, `core.hooksPath` unset |
| **No coverage measurement** (MEDIUM) | **STILL OPEN** | a new untested export still exits 0 on both gates (see below) |

Two things follow, and they are the real content of this revision.

**1. The forged-suite fix is two independent checks, and both are needed.**
`2934001`/`a494839` added (a) a cross-check that the reported pass count matches the
assertion lines actually emitted, and (b) a V8 coverage gate — a suite must execute at least
one `src/` file. The commit message documents that (a) alone is insufficient: with
`POLYMEM_SKIP_COVERAGE_GATE=1` the same forgery still passes. That is a correct piece of
engineering and it is worth stating plainly here, because a green run now means more than it
did — it means *both* an emitted-line count agreed *and* library code was executed.

**2. The root cause is still not fixed.** The coverage gate asks *"did this suite touch
`src/` at all?"*, not *"does every exported function get exercised?"*. Measured at `2be9fc2`:

```
appended an untested exported function to src/polymem.mjs
npm test        -> exit 0   17 suites, 559 assertions, 0 failing
npm run docs:check -> exit 0   All 46 doc claims match measured reality
```

Both gates green on code that no suite calls and nothing covers. The floors count
**assertions a suite reports**, never **source lines exercised**. This is the row that the
original audit called the root cause of GAP 1, and it survives the fix to GAP 1.

The honest summary of `2be9fc2`: the trust model is materially stronger than it was at
`bdce4bb`, the HIGH gap is closed, and the structural limit that made the HIGH gap possible
is unchanged.

---

## 1. What was measured, and how

Everything below was run by hand. No result is inherited.

| | |
|---|---|
| Repo | `Polymem-clean`, committed HEAD `bdce4bb`, clean working tree |
| `npm test` | `16 suites, 548 assertions, 0 failing` — **exit 0** |
| `npm run docs:check` | `All 46 doc claims match measured reality` / `548 assertions / 16 suites, 7 packed files` — **exit 0** |
| Floors in CI sentence | `440 assertions / 14 suites` vs measured `548 / 16` |
| Tracked files PII-scanned | 63 at `bdce4bb`; **64 once this file is committed** |
| Commit objects authorship-scanned | 5, all reachable from a ref |

The older figure `508 assertions / 15 suites` belongs to `022a0c6` and is **stale**. Any
document still quoting it is describing a tree that no longer exists.

On the tracked-file count: this document is itself a tracked file, and the PII scan reads
every tracked file including this one. Committing it takes the count from 63 to 64. **That
is expected, and it is the guard working** — a reader who sees 64 has not found drift. This
paragraph exists so the number is not mistaken for a stale claim later, which is the same
trap `docs/2026-09-25-promotion-defect.md` records at length when its own suite-size figure
outlived its truth.

### Method

Probes ran in a **real `git clone`**, never `git archive` into `/tmp`, and each probe had a
**control**: the same clone, unaltered, run first.

This is not fussiness. `docs:check` derives its file list from `git ls-files`, so in a bare
`git archive` tree it exits 1 *for that reason alone* — on a clean, un-forged tree too. That
red looks exactly like a guard catching a forgery and it is the opposite. Verified both ways:
the un-forged control exits 0 in a clone, and the control was what made the GAP 1 result
below trustworthy. **A red that fails for a non-defect reason is not a red.**

---

## 2. Per-guard evidence

Each row was proven by planting the violation and reading the real exit code. Not inferred
from the YAML, not inferred from the source.

| # | Guard | Violation planted | Result |
|---|---|---|---|
| 1 | PII — email | email address appended to a tracked file | **exit 1** `✗ no personal data: README.md` |
| 2 | PII — home path | absolute macOS home path in a tracked file | **exit 1** `✗ no personal data: README.md` |
| 3 | PII — phone | NANP phone number in a tracked file | **exit 1** `✗ no personal data: README.md` |
| 4 | Commit authorship | commit by a non-noreply address | **exit 1**, author **and** committer named separately |
| 5 | Doc-claim drift | a suite deleted, so the documented count no longer matches | **exit 1** `✗ README.md Status states the measured assertion/suite count` |
| 6 | Suite emptied | suite truncated to 0 bytes (emits no result line) | **exit 1** |
| 7 | Unregistered suite | new suite file with no floor row | **fatal**, run refuses to proceed |
| 8 | CI floor sentence | `.github/workflows/ci.yml` floor vs `test/suite-floors.mjs` | green, `ci.yml says 440/14, suite-floors.mjs says 440/14` |
| 9 | Tarball contents | `npm pack --dry-run` vs expected public file list | green, 7 files, no `test/ docs/ sim/ tools/` |
| 10 | Green-at-HEAD | doc guard over a red suite | green, `0 failing across 16 suites` |
| 11 | **Forged result line** *(new at `2be9fc2`)* | suite prints a result line, imports nothing from `src/` | **exit 1** — `reported pass=5 but emitted 0 passing assertion line(s)` |
| 12 | **Suite never touched `src/`** *(new at `2be9fc2`)* | same forgery, coverage cross-check on | **exit 1** — V8 profile has no `src/` file |

Guards 11 and 12 did not exist at `bdce4bb`; they are the fix for GAP 1, and the note in
the GAP 1 commit message is that 12 is load-bearing on its own. Guards 1–10 were
re-confirmed unchanged at `2be9fc2`.

The PII scan is **fail-closed and derived, not hand-listed**. It enumerates
`git ls-files` and refuses to certify a scan it could not read — including the case where
git reports zero files, which is treated as an error rather than as "clean". This is the
right shape: an earlier hand-written 13-entry list could not see 48 of 61 tracked files, and
it rotted *silently* — the check kept passing, so nothing announced the decay.

Commit authorship is read from the **whole object database**
(`git cat-file --batch-all-objects`), not from `git log --all`. A commit no ref points at
still exists, can still be pushed by SHA, and `--all` will never report it. Off-ref
commits are called out separately in the failure message because they have a different fix
(`reflog expire` + `gc`) than a published one.

### CI wiring

`.github/workflows/ci.yml` runs `npm test` on Node 18 and 20, and `npm run docs:check` on
Node 20 only (it shells out to the suite itself, so running it twice would double CI time
to re-derive a number that cannot differ between the two). Triggers: `push` to
`main`/`master`, `pull_request`, `workflow_dispatch`. A non-zero exit from a `run:` step
fails the job, so no extra step is needed to honour it.

There is **no pre-commit hook** (`.git/hooks/` holds only samples; `core.hooksPath` is
unset). CI is therefore the *first* place any of this runs. See gap 4 in §5.

### Fresh-repo check

A workflow that fails on a first push is the worst possible first impression on a public
repo, so this was tested on a single-commit tree with no remotes and no tags:

```
commits=1  remotes=0  tags=0
npm test        -> exit 0   OK — 548 assertions across 16 suites
npm run docs:check -> exit 0  All 46 doc claims match measured reality
```

Nothing in the workflow assumes history, tags, or a release. It is clean on a fresh repo.

---

## 3. GAP 1 — HIGH — a suite can forge its own result line

**Still open. Reproduced at `bdce4bb`.**

`test/run-tests.mjs` reads the assertion count from a line **the suite itself prints**, and
believes it. Replace any suite with three lines that assert nothing at all:

```js
import { RESULT_PREFIX } from './harness.mjs';
console.log(RESULT_PREFIX + ' ' + JSON.stringify({
  name: 'test-repo-root-containment.mjs', pass: 5, fail: 0, floor: 5, floorOk: true
}));
```

Measured, in a real clone, with the un-forged control run first:

```
CONTROL  (un-forged)   npm test exit 0    docs:check exit 0    "16 suites, 548 assertions, 0 failing"
FORGED                npm test exit 0    docs:check exit 0    "16 suites, 548 assertions, 0 failing"
grep -c "src/" test/test-repo-root-containment.mjs  ->  0
```

**Both CI steps go green on a suite that exercises nothing.** The reported count is
unchanged because the runner reads it from the forgery. The suite no longer imports the
library at all.

`test/suite-floors.mjs` states the invariant it believes it is enforcing:

> At the exact count, the ONLY edit that keeps the suite green is ADDING assertions.

That invariant is not enforced. The floors are real — they catch accidental gutting
(guards 6 and 7 above) — but they do not catch a suite that *lies*.

**The asymmetry worth naming:** every accidental failure mode is caught. Truncating a
suite, removing its assertions, adding an unregistered suite, deleting suites — all caught.
Only the *deliberate* rewrite passes. That is a real limit, not a rounding error, and a
future table-driven suite that prints result lines while delegating its counting would
reopen it.

**What would close it:** have the runner cross-check `reported.pass` against the number of
assertion result lines the suite actually emitted, and fail on disagreement. A few lines in
`harness.mjs` + `run-tests.mjs`, plus a test that forges the line and asserts exit 1. That
defeats the forgery; it does not make the harness forgery-proof, and the commit message
should say so rather than implying otherwise.

---

## 4. GAP 2 — MEDIUM — the suite-count floor has slack, and it has grown

**Still open, and worse than previously reported.**

`SUITE_COUNT_FLOOR = 14`, `TOTAL_ASSERTION_FLOOR = 440`, measured `16 / 548`.

The earlier audit reported slack of exactly 1 at `022a0c6` (15 measured, floor 14), and
correctly noted that deleting one suite landed on `14 >= 14` and passed. **Since then two
suites were added and the floor was not raised**, so the slack is now 2:

| Deleted | `npm test` | measured | `docs:check` |
|---|---|---|---|
| 1 suite | **exit 0** | 15 suites / 543 assertions | exit 1 |
| **2 suites** | **exit 0** | **14 suites / 529 assertions** | exit 1 |
| 3 suites | exit 1 | 13 suites / 521 assertions | exit 1 |

The earlier note that "deleting **two** does fire (13 < 14)" was true at `022a0c6` and is
**false at `bdce4bb`**. Two whole suites can now be deleted and `npm test` is green.

CI is still covered: `docs:check` catches all three cases by comparing the measured counts
against the documented ones. But `npm test` alone is what a human runs before pushing —
which is the failure this whole workstream exists to prevent.

**A floor that rots upward is worse than no floor**, because it reads as protection. Each
suite added without raising the floor widens the hole silently, and nothing announces it.

**GAP 2b — still open.** `test/run-tests.mjs:74` checks one direction only: a suite with no
floor row (`suites.filter(n => SUITE_FLOORS[n] === undefined)`). It never checks the
reverse — a floor row whose suite file was deleted. The stale row then sits in
`suite-floors.mjs` certifying coverage that no longer exists. Measured: delete a suite
*and* its floor row, and `npm test` reports `15 suites, 543 assertions, 0 failing`, exit 0.

**What would close it:** set `SUITE_COUNT_FLOOR` to the real suite count so slack is zero;
add the inverse check so a leftover row is detected; update the `.github/workflows/ci.yml`
floor sentence in the **same commit** — `docs:check` §5 verifies that sentence against
`suite-floors.mjs`, so changing one without the other fails the guard.

---

## 5. GAP 3 — already fixed at HEAD

**The earlier report's GAP 3 does not reproduce. The code it describes is gone.**

The reported defect was a `git log --all --format=%an|%ae` call with no `cwd: REPO`, so the
check read whatever repository the process happened to be sitting in, with `.stdout || ''`
turning a *failed* read into an empty string that then reported success.

At `bdce4bb` the old form is **absent** — the literal `--format=%an|%ae` appears nowhere in
`tools/check-doc-claims.mjs`. That whole block was replaced. It now:

- reads the **object database** (`git cat-file --batch-all-objects`, line 702) with
  `cwd: REPO`, not a ref walk;
- passes `cwd: REPO` to **every** git invocation in the file, and throws on a non-zero
  status rather than coercing it to empty (lines 707, 735);
- enumerates author **and** committer from an explicit `--stdin` revision list, so nothing
  is filtered out in transit (line 730);
- treats **zero** enumerated commits as an error, not as a clean history (line 720).

The failure mode was also re-tested directly. The original probe ran the checker from a
non-repository directory and expected a miss:

```
cwd = the repo       -> exit 1   caught (correct)
cwd = a non-repo dir -> exit 1   CAUGHT — the earlier report predicted a miss here
```

From a non-repo cwd it still names all four bad identities and still fails. There is no
longer a path by which an unreadable history is reported as clean.

Card `t_a7e64439` should be closed as already-resolved. Its premise was accurate when
written; the code moved underneath it.

---

## 6. The coverage gap — what these checks do NOT catch

The card asked for this stated plainly, and it is the most important section here.

> **Updated at `2be9fc2`.** Three of the four rows below have been fixed since this was
> measured; they are kept because they show *what the fix had to beat*. The remaining row was
> re-verified at `2be9fc2` and still passes green.

**These checks verify their own assertions. A suite that lies about what it asserts defeats
every one of them.** That was GAP 1, and it is why the trust-model gap outranked the other
two: the other two are arithmetic, this one is the premise the arithmetic rests on. At
`2be9fc2` the *premise* is defended — a lying suite can no longer get both gates green — but
the *premise itself* (that the suites exercise the library) is still taken on faith.

Concretely, all of the following were run, and the result column is the state at `bdce4bb`
followed by the state at `2be9fc2`:

| Class | `bdce4bb` | `2be9fc2` |
|---|---|---|
| A suite forged to assert nothing (GAP 1) | `npm test` exit 0, `docs:check` exit 0 | **exit 1 / exit 1 — fixed** |
| A brand-new exported function in `src/`, untested and arbitrary | `npm test` exit 0, `docs:check` exit 0 | **still exit 0 / exit 0** |
| Up to two entire suites deleted (GAP 2) | `npm test` exit 0 | **exit 1 — fixed** (zero slack) |
| A suite deleted *and* its floor row removed (GAP 2b) | `npm test` exit 0 | **exit 1 — fixed** (inverse check) |

The surviving row deserves emphasis. The floors count **assertions a suite reports**, never
**source lines actually exercised**. The coverage gate added for GAP 1 closes the gap at the
*suite* granularity — "this suite touched `src/` at least once" — and that is the right level
to catch a forgery, but it is not per-export. There is still no coverage measurement in this
repository and no dependency that could provide one. So a new exported function can be added
that no suite calls, no assertion can fail, and the total stays at 559 — because the total is a
count of *claims*, not of *coverage*. A green suite here means "every assertion that was
written passed, and every suite touched the library", which is stronger than it was and still
a strictly weaker statement than "the library works".

Two further limits, both structural:

- **PII inside non-text files is not guaranteed visible.** The guard decodes binary files as
  UTF-8 and says so in its own output when it does. An address inside a real binary would not
  be caught.
- **Nothing runs before the push.** There is no pre-commit hook. The sequence is
  *commit → push → CI*. The failure this workstream was created after — a push that went out
  while a check was failing — is caught by CI, but only *after* the commit is public. A
  `pre-commit` hook running `npm test` and `docs:check` would move the gate to before the
  push, which is where a guard belongs. This is the cheapest item on the list and it is not
  on anyone's card.

### A trap that fires on the committer, not just the author

The machine's global git identity on this workstation is **not** a noreply address. A commit
made without an explicit per-commit identity is therefore rejected by the authorship guard —
and not only as author. `git commit --amend --reset-author` fixes the **author** while
leaving the **committer** as the offending address, so the guard still fails and the message
still names the same SHA. Every commit in this repo's history carries the maintainer's
noreply address on **both** sides; that is the shape to match. (It is spelled from
fragments here for the same reason the guard assembles its own allowlist from fragments:
this file is tracked, and a literal address would trip the PII scan on its own audit.)

Amending does not make the bad object disappear. The guard reads the **whole object
database**, so the pre-amend commit survives as an unreachable object and keeps failing the
check until it is pruned. The guard names this itself — `off-ref: gc will remove it` — and
the fix is the pair:

```bash
git reflog expire --expire=now --all && git gc --prune=now
```

Run that only after confirming `git fsck --unreachable` lists **nothing but the object you
just orphaned**. On this repo it listed exactly one commit and one blob, both from the
identity mistake, and no stashes existed. Anything else in that list belongs to someone
else's in-flight work and pruning it would destroy their work silently.

### What is genuinely true, and what is not

A team that believes its checks are comprehensive is worse off than one that knows its limits.

What is genuinely true: file-content PII, commit authorship (including off-ref objects),
doc/reality drift, accidental suite damage, unregistered suites, and tarball contents are
all covered, all fail-closed, and all wired into CI on both Node versions.

What is not true, and should not be read into the green: that the suite exercises the
library, that the assertion count reflects coverage, that a hostile or careless suite cannot
report a number it did not earn, or that anything at all runs before a push leaves the
machine.

---

## 7. Corrections to the earlier record

Carried here so the next reader does not re-import the wrong numbers:

1. **`508 assertions / 15 suites` is stale.** It describes `022a0c6`. HEAD `bdce4bb` is
   **548 / 16**, and `docs:check` now checks **46** doc claims, not 44.
2. **GAP 3 does not exist at HEAD.** Fixed, verified by re-running the exact failure mode.
   `t_a7e64439` is stale.
3. **GAP 2 is worse than reported.** Slack was 1, now 2. Deleting two suites is green at
   `bdce4bb`; that was not true at `022a0c6`.
4. **`10_BRAIN/runbooks/polymem-harness-gate-2026-10-01.md` never existed.** No `10_BRAIN/`
   tree in either local clone, any branch, or any stash. This file is the real artifact.

---

## 8. Reproduce this audit

Figures below are the `bdce4bb` baseline. At `2be9fc2` the expected results for the GAP 1 and
GAP 2 probes have **changed**: each of those lines now prints `EXIT=1`, which is the correct
answer. That change is the fix landing, not the probe breaking.

```bash
# green at HEAD
npm test && npm run docs:check

# GAP 1 — forge a suite. Use a real clone; `git archive` makes docs:check exit 1
# for a reason that has nothing to do with the forgery.
D=$(mktemp -d); git clone . "$D"; cd "$D"
cat > test/test-repo-root-containment.mjs <<'EOF'
import { RESULT_PREFIX } from './harness.mjs';
console.log(RESULT_PREFIX + ' ' + JSON.stringify({
  name: 'test-repo-root-containment.mjs', pass: 5, fail: 0, floor: 5, floorOk: true
}));
EOF
npm test; echo "EXIT=$?"                # bdce4bb: 0 — should be 1 / 2be9fc2: 1
npm run docs:check; echo "EXIT=$?"       # bdce4bb: 0 — should be 1 / 2be9fc2: 1

# GAP 2 — two suites, not one (zero slack at 2be9fc2)
rm test/test-repo-root-containment.mjs test/test-session-date-containment.mjs
npm test; echo "EXIT=$?"                # bdce4bb: 0 — should be 1 / 2be9fc2: 1
```

**Reproduction trap, unchanged and still worth stating.** Use `git clone`, never `git archive`
into `/tmp`. `docs:check` derives its file list from `git ls-files`, so a bare archive tree
fails for layout reasons alone — on a clean, un-forged tree too, and it does so with the same
red text a caught forgery produces. **A red that fails for a non-defect reason is not a red.**
This trap also fires on the fresh-repo check in §2: a synthetic test commit whose identity is
not the allowlisted noreply address fails `docs:check` on authorship alone, which says nothing
about history-dependence. Build the fresh repo with the allowlisted identity on **both** author
and committer before drawing any conclusion.

The planted-violation probes in §2 append a fake address to a tracked file. The literal
values are assembled from fragments in the shell rather than written out, because this
document is itself a tracked file and the PII scan reads it like any other — a literal
address here would trip the guard on its own audit.

---

## 9. Open items

Re-verified at `2be9fc2`. Two items remain genuinely open.

| Gap | Severity | State at `2be9fc2` | Card |
|---|---|---|---|
| Forged suite result line | HIGH | **CLOSED** — two independent checks, both required | `t_1b513db1` (close as resolved) |
| Suite-count floor slack + missing inverse check | MEDIUM | **CLOSED** — floors pinned to 17/559, zero slack | `t_4a43f18c` (close as resolved) |
| Commit-authorship guard reads the wrong repo | — | **already fixed** before this revision | `t_a7e64439` (close as resolved) |
| No pre-commit hook — nothing runs before the push | MEDIUM | **open, unfiled** | — |
| No coverage measurement — a new untested export is invisible | MEDIUM | **open, unfiled** | — |

All three filed gaps are closed. The two unfiled ones are not, and neither will turn a launch
red — but they are the difference between "our checks pass" and "our checks mean something".

The pre-commit hook is now the cheapest remaining item *and* the most urgent, because the
repo has gone public (commit `7c3016e`): a commit pushed to a public origin with a failing
check is visible before CI reports anything, and the only thing that would have caught it
earlier does not exist locally.

The coverage-measurement item is the root cause of GAP 1 and it is still open. GAP 1 is
closed against the *known* forgery shape, not against the class: the new gate is per-suite,
so a suite that genuinely touches one `src/` file while ignoring the rest of the library
still passes. Closing the class means per-export coverage, which means a coverage tool — a
dependency the package currently does not have, and a deliberate call for whoever owns it.
