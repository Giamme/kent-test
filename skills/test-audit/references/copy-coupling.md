# Focused audit: copy-coupled tests

Invoked as `/test-audit copy [path]`. It hunts one junk pattern, tests that
break whenever wording changes, and nothing else. Everything in `SKILL.md`
still applies (baseline, ledger, proof, edit shape, validate, report), narrowed
as below.

## 1. Scope and baseline

- Scope is the path given, otherwise every test that renders UI or inspects
  rendered output: component, page, end-to-end, snapshot, and response-body
  tests.
- Baseline as in `SKILL.md` section 1.
- Ledger only copy-coupled candidates. Do not ledger, edit, or delete other
  junk you notice; list it as a one-line follow-up ("also saw: ...").

## 2. Discovery

Search the scope for tests that match or assert prose by literal:

- Testing Library: `getByText`, `findByText`, `queryByText`, `getByLabelText`,
  `getByPlaceholderText`, `getByTitle`, `getByRole(..., { name: "..." })`,
  `toHaveTextContent`, `toHaveAccessibleName`.
- Playwright, Cypress, WebdriverIO: `getByText`, `locator("text=...")`,
  `toHaveText`, `cy.contains`, `$("=...")`.
- Snapshots (`toMatchSnapshot`, `toMatchInlineSnapshot`, snapshot files) of
  rendered output that includes copy.
- Server and CLI tests that assert prose in a response body or output
  (`assert "Welcome back" in html`, `expect(out).toContain("...")`).

Classify each hit:

- **Outcome assertion**: the text is the evidence that something worked
  (`expect(getByText("Saved!")).toBeVisible()` as the only proof of a save).
  This is the main target.
- **Locator**: the text only finds an element to act on. Lower priority. Fix it
  only when the repo already queries by role or test id, or when one label is
  repeated across many tests. Otherwise leave it. Role plus accessible name is
  legitimate when the accessible name is itself what is being verified.
- **Contract**: the copy is pinned on purpose: legal or compliance wording, a
  localization key, an error code or message consumers match on, text
  documented as stable. Keep, and record the contract in the ledger.
- **Snapshot with copy**: it breaks on any wording change but may guard layout
  or structure too. Judge it by what else it protects; usually fix, replacing
  it with targeted assertions, and never drop it without a keeper.

## 3. Probe: change the copy

This replaces the generic mutation for candidates.

1. Find the string the test depends on in production code or the message
   catalog. Reword it without changing its meaning ("Save" to "Save changes").
2. Run the candidate. If it fails, it is copy-coupled: confirmed. If it passes,
   the text comes from the same source or the match is loose, so it is not
   coupled; remove it from the ledger.
3. After repairing, reword again and confirm the test passes. Then make one
   behavior mutation (remove the handler, skip the state update) and confirm
   the repaired test fails. Both are required: a repair that survives a reword
   and also survives a broken behavior is only a weaker test.
4. Restore the production file byte for byte and check `git diff` is empty for
   it. If a watcher or dev server runs from the checkout, or the user asked for
   a report only, probe in a throwaway `git worktree`.

One reworded string covers every test that uses it; record which ledger rows
each probe covers. A test that imports the same message or catalog entry as the
component is not copy-coupled.

## 4. Verdicts and repairs

- **fix** (the usual verdict): replace the literal with behavior. Find the
  element by role or label the UI already exposes, then assert the state
  change, navigation, ARIA state, presence or absence, or count. If text must
  be asserted, derive it from the same catalog entry the component uses.
- **drop or merge**: the test only proves "renders text X" and another test
  already renders the component and exercises its behavior. Name that keeper and
  prove it (`SKILL.md` section 4).
- **keep**: a contract.

A looser match that still contains the same words (a regex, a substring) is not
a repair: it breaks on the next rewording. Do not add test ids to production
code for this unless the repo already uses them; a test-only attribute is a
seam (test-gate section 2, question 4).

## 5. Report

The `SKILL.md` report, narrowed to this pattern, plus: the strings you probed
and the ledger rows each probe covered, and how many copy-coupled tests were
found, fixed, dropped or merged, and kept as contracts.
