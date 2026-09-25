# Campaign: prune a whole subsystem's tests

A campaign audits every test a subsystem owns (a package, a plugin, a core
area) in one change. Everything in `SKILL.md` still applies to every part of
it; this file adds the order of work. Finish each step before starting the
next.

## 1. Baseline

Pin a commit. Record every in-scope test file's pass/fail state and the line
counts of test code, test support, and production code. Keep baseline failures
on their own list; in the campaign this is adapted from, every baseline failure
turned out to be a real product bug, not a stale test.

Done when every in-scope test file has a recorded result.

## 2. Lanes

Split the surface into lanes along production ownership (accounts, transport,
persistence, commands, and so on), not along file-name prefixes. Include the
subsystem's cases in shared suites and its end-to-end or QA scenarios.

Done when every test file and scenario belongs to exactly one lane.

## 3. Ledger per lane

Give each lane to its own read-only subagent. It reads every assigned test in
full, including parameter tables, plus the production owners, their callers,
history, and CI routing, and returns ledger rows (`SKILL.md` section 3) with a
one-line evidence note each. Judge by assertions, not names: one test named for
retiring a window asserted the window was *not* cleared.

Done when every test in the lane has a verdict and evidence.

## 4. Layer plan per lane

Treat the ledger as input, not the edit list. A second read-only pass looks
for redundant layers: several suites replaying the same shared logic through a
mocked collaborator beside a stronger suite that uses the real boundary with a
fake network or store. Name the keeper for each contract; prefer the real
boundary over mocked collaborators. Fix ledger mistakes this pass finds.

Done when each lane plan names its retired files, the keeper per contract, the
assertions to carry into keepers, and the test-only seams unlocked.

## 5. Cutover

Edit lane by lane. Route changes to shared harnesses and support files through
one owner so lanes do not collide. With each lane, remove the test-only seams
it unlocks. Register moved suites wherever the repo lists tests (CI routing,
inventories). If the user wants durable rules, add test-ownership notes to the
subsystem's `CLAUDE.md` or `AGENTS.md`, drawn from mistakes the campaign
actually found.

Done when every lane plan is applied and each lane's keepers pass.

## 6. Preservation review

Before claiming completion, have independent reviewers (one subagent per
boundary group) compare the deleted coverage against the keepers. They look
for contracts that lost their only proof, and for new assertions that cannot
fail, such as a rejection row the production code never reaches. For each
restored contract, make one deliberate mutation of the production owner,
confirm the keeper fails, and restore the source byte for byte.

Done when every reported gap is restored or rejected with source evidence.

## 7. Product defects

A baseline failure that survives into a keeper is a bug. Fix it at its owner
as a separate change, and prove it with a control run: revert the fix, show the
old behavior, restore it. Record unrelated discrepancies as follow-ups instead
of fixing them inside the campaign.

## 8. Reconcile and hand off

Campaigns outlive many commits on the main branch. Merge main rather than
rebasing a long campaign. When main changed a file the campaign deleted, keep
the deletion and port the new contract into the keeper; confirm every
regression test main added still has a home. Rerun the whole subsystem suite on
the merged result.

Hand off with the `SKILL.md` report plus: baseline and final line counts
(production separately), lanes, retired layers and keepers, preservation gaps
and their mutations, and product defects with control and fixed runs.
