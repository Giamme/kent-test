---
name: test-audit
description: Audit existing tests for ones that cost more than they protect (duplicated coverage, implementation-coupled assertions, tests that cannot fail, mocks that supply the answer, test-only production seams) and prune or repair them without losing any contract's only proof. Use when the user asks to audit, prune, clean up, dedupe, slim down, or sweep a test file, module, directory, or subsystem, or says the tests are bloated, brittle, or low value and wants that fixed. Not for writing a new test or reviewing a single fresh test change; the test-gate skill covers those. Pass `copy` to audit only for tests coupled to UI wording, and also use that mode when the user says tests break whenever copy or labels change.
argument-hint: "[copy] [path]"
---

# Test Audit

An audit removes or repairs tests that do not earn their maintenance cost, and
it must not lose protection while doing so. Optimize for confidence, not for
deletion count: a short list of proven removals beats a long speculative one,
and "this suite is healthy" is a legitimate finding.

Repairs weigh as much as removals. A test that claims a behavior its
assertions do not enforce gives false confidence, which is worse than having
no test, so finding and fixing such tests is often the most valuable part of an
audit. Do this even when the user only said "prune" or "cut": a weak test that
guards a real contract gets repaired, not deleted and not left alone.

The value bar, the junk patterns, and the keep / fix / merge / drop verdicts
live in the **test-gate** skill (`../test-gate/SKILL.md`, sections 2, 3 and
6). Read them before judging anything. This skill adds what an audit needs on
top: a baseline, evidence for each candidate, proof that removals lose nothing,
and the shape of the edit.

If test-gate is not installed next to this skill, apply this minimum instead.
A test earns its place when it protects behavior a caller or user relies on,
fails on a credible regression, and is not already covered at a stronger
boundary. Junk includes:

- tests with no assertion;
- expected values produced by the code under test;
- copied inventories of constants or fixtures;
- source-text greps;
- assertions on exact UI copy (labels, headings, messages) when the behavior
  is what matters;
- mocks that return the value being asserted;
- duplicated scenarios;
- negative cases that pass for the wrong reason;
- names that promise more than the input exercises.

Verdicts are keep, fix, merge, or drop.

**Scale.** The default is a focused audit of one file, module, or directory.
For a whole subsystem's test surface in one change, read
`references/campaign.md` first and follow it instead.

**Focus.** If invoked with the argument `copy` (`/test-audit copy [path]`), or
the user asks only about tests that break when wording changes, audit for that
one pattern: read `references/copy-coupling.md` and follow it instead of the
general discovery below. The rest of the argument is the scope.

**Audit versus cleanup.** If the user asked for an audit or a report, stop
after the ledger (section 3) and let them choose. If they asked you to clean up
or prune, carry on through the edits, but the ledger still leads the report.

## 1. Baseline

Before judging any test:

- Learn how the repo runs tests (test-gate section 1): docs, manifests, the
  narrowest command for the scope.
- Run the in-scope tests and record the commit, the pass/fail state, and the
  line counts of test code, test support code, and production code separately.
- A test that fails on the baseline is not a stale test to delete. Treat it as
  a possible product bug: keep it on its own list and investigate.
- Make sure no watch-mode test runner is running in the checkout, because it
  re-runs half-edited files and muddles results.

## 2. Discovery

Stay read-only until the ledger exists. For every test in scope, read the whole
test and the production code it exercises: the entry point, its callers, the
sibling tests, and the test's history (`git log --follow -p` on the test file,
and the commit that introduced the test). A test added alongside a bug fix is a
regression test; find out which bug before judging it.

Hunt for the test-gate junk patterns, plus what only shows up across a suite:

- **Redundant layers**: the same contract asserted in a unit test, a component
  test, and an integration test. Keep the strongest boundary that can observe
  it; keep another layer only for a risk the first cannot reach.
- **Test-only production seams**: exports, reset functions, injection
  parameters, getters, globals, or wrappers with no caller outside tests.
  Search for callers across the repo, excluding test files, to confirm.
- **Suspicious coupling**: tests that would have to change for a
  behavior-preserving refactor. Suspect, not automatically deletable: prove it
  (section 4).
- **Assertions that cannot fail**: rejection cases the production code never
  reaches, negative controls rejected by a different guard, names that promise
  more than the input exercises.
- **Mock drift**: a mock whose shape or behavior no longer matches the real
  dependency, so the test passes against something that does not exist.
- **Copy-coupled assertions**: tests that fail when only the wording of a
  label, heading, or message changes. Prove it by rewording the string and
  rerunning, as in `references/copy-coupling.md` section 3.

For a large scope, split discovery into areas along production boundaries and
give each to a read-only subagent that returns ledger rows; merge and check the
rows yourself.

Beware false positives. A test that restates a list or a string can still be
the only proof of a contract: user-visible order, a wire format, a documented
limit, a config default, a migration, a security rule, legal or localization
copy, an error message consumers match on. Before calling it junk,
ask what regression would slip through without it.

## 3. The ledger

Record one row per test (a parameterized test is one row unless its cases need
different verdicts):

| Field | Content |
|---|---|
| Test | name and location |
| Verdict | keep, fix, merge, or drop |
| Detects | the failure it can actually catch, judged from its assertions, not its name |
| Keeper | for merge or drop: the test that still proves the contract, or why no contract exists |
| Seams | production code that only this test uses, and what removing the test unlocks |
| History | why it was added, if the history says |
| Proof | the mutation result from section 4, or "unproven" |

A drop or merge with an empty Keeper or Proof field is not ready. Either prove
it, or downgrade it to keep or fix and say what is uncertain.

## 4. Prove that removals lose nothing

For every drop or merge, make one deliberate change to the production code that
breaks the behavior the test claims to protect (delete the guard, flip the
comparison, move the limit by one). Then:

1. Run the keeper and confirm it fails. Running the candidate too shows the two
   really cover the same thing.
2. If no keeper fails, the candidate is the only proof. Keep it or fix it.
3. For a merge, run the mutation again after the edit and confirm the carried
   assertion fails in its new home.
4. Restore the production file byte for byte and check `git diff` is empty for
   it.

The same probes find weak tests. When a mutation survives even though some
test's name, description, or apparent purpose claims to cover that behavior,
give that test a **fix** verdict and say exactly what to change: the input that
would exercise the path, or the assertion that would catch it. It is not a
footnote under follow-ups. Only behavior that no test claims to cover is a new
coverage gap for the follow-ups list.

A test that demonstrably cannot fail (no assertion, comparing a function with
itself, a mock returning the asserted value) needs no mutation: show why in the
Detects field. When a mutation is impractical, say so and keep the test unless
it cannot fail.

Temporary mutations must not disturb anyone. If the user asked for a report
only, or a dev server or watcher may be running from the checkout (hot reload
would pick up the broken file), run the probes in a throwaway `git worktree`
instead of the user's checkout.

## 5. Edit shape

- One coherent batch per production owner. Do not mix unrelated areas.
- Delete the test-only seams the removals unlock (exports, reset hooks,
  injection parameters, wrappers, and dead production code whose only callers
  were tests) instead of leaving compatibility aliases. Fewer production lines
  is the goal.
- Move a retained regression test to the test file of the module that owns the
  behavior, and fold near-duplicates into one table.
- Repair a fix-verdict test so it fails for the intended reason (test-gate
  section 3), rather than deleting it.
- Do not add replacement tests that restate the implementation, and do not turn
  uncertain candidates into deletions to raise the count.
- A baseline failure that is a real product bug gets reported. Fix it only if
  the user asks, as a separate change.

## 6. Validate

- Run the owner's tests and its siblings, then the repo's checks for the
  changed files (lint, typecheck, formatter).
- If you removed a source-inspection test, run whatever real check owns that
  contract.
- Compare line counts against the baseline, production separately from tests.
- Commit, push, or open a PR only when asked.

## 7. Report

- scope, baseline commit, and baseline failures;
- the ledger, with the exact change for each fix verdict;
- the mutation proofs for each drop and merge;
- production seams removed;
- tests that looked like junk but were kept, and the contract each guards;
- commands run and results;
- line counts before and after, production and tests separately;
- follow-ups: product bugs found, the next area worth auditing.

---

Adapted from the `test-audit` skill in
[OpenClaw](https://github.com/openclaw/openclaw) (`.agents/skills/test-audit`),
Copyright (c) 2026 OpenClaw Foundation, MIT License. See `LICENSE-openclaw`.
