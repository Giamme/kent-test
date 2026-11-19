# Test conventions: agree them once per repo

A test can be worth keeping and still be wrong for the team. That happens when
it sits in a folder nobody uses, follows a naming scheme the suite doesn't
use, builds its own mocks next to the shared ones, or lands in a file the
runner or CI never picks up. Reviewers reject that work however good the
assertions are. Most of these rules are never written down: they live in the
existing tests and in past review comments.

So the first time either skill runs in a repo, before any test is written or
moved, survey how the team tests, agree the conventions with the developer, and
save them where the next session will find them. After that, read the saved
record instead of surveying again.

## 1. Find the record

The record is a Markdown section or file that has the marker
`<!-- kent-test:conventions -->` on a line of its own. Look in two places:

- in the repo:

  ```sh
  git grep -lE --untracked '^<!-- kent-test:conventions -->$' -- ':!*test-gate/*'
  ```

  The pathspec skips copies of this skill that a repo may vendor. A team that
  shares the record usually keeps it in `AGENTS.md`, `CLAUDE.md`,
  `CONTRIBUTING.md`, `TESTING.md`, or a `docs/` page;
- in this clone only:
  `"$(git rev-parse --path-format=absolute --git-common-dir)/kent-test/conventions.md"`.
  It sits inside `.git`, so it is never committed, and every worktree of the
  clone shares it.

If both exist, the one in the repo wins. Mention the difference to the
developer.

- **Record found:** read it, then go to section 5.
- **No record:** run sections 2 to 4 before writing or moving any test.
- **The user asks to set up, review, or redo the conventions** (`/test-gate
  conventions`, "redo the test conventions"): run sections 2 to 4 even if a
  record exists, starting from it.

This runs once per repo, so do it even when the task is small. It should take
minutes, not an hour. If the developer says "not now", follow the dominant
observed patterns (section 6), say which ones you assumed, and ask again in the
next session, not in this one.

## 2. Survey

Measure. Do not guess from one sibling file. Stay read-only.

1. **Written rules.** Read the root and nested `AGENTS.md`, `CLAUDE.md`,
   `CONTRIBUTING.md`, any `docs/` page about testing, the PR template, lint
   configs with test rules (`eslint-plugin-jest`, `eslint-plugin-testing-library`,
   `ruff` test rules, `clippy` settings), and `CODEOWNERS` entries for test
   paths. Written rules are the conventions unless the code clearly contradicts
   them; then ask.
2. **Where tests live.** List every test file and group it by layout, per
   package or top-level area. A monorepo often has different rules per package.
   A starting filter:

   ```sh
   git ls-files | grep -E '(\.|_)(test|spec)s?\.[a-z]+$|(^|/)(tests?|__tests__|spec|e2e|integration)/|(^|/)test_[^/]+\.py$|_test\.go$'
   ```

   Tell apart the layouts that actually occur: next to the source
   (`foo.test.ts` beside `foo.ts`), a `__tests__/` folder per directory, a
   top-level `tests/` mirroring `src/`, Rust `mod tests` inside the module
   versus `tests/*.rs`, a separate `e2e/` or `integration/` tree. Count files
   per layout and keep two example paths of each.
3. **Kinds and their homes.** Find which kinds of test the repo has (unit,
   component, integration, end-to-end, contract, snapshot, benchmark), where
   each kind lives, and which command and CI job runs it. Read the runner
   config's include and exclude globs (`vitest.config`, `jest.config`,
   `playwright.config`, `pytest.ini` or `pyproject.toml` `testpaths`, Cargo
   test targets) and the CI workflows. A test outside those globs never runs,
   even if it passes on your machine.
4. **Recent practice.** List where tests were added recently:

   ```sh
   git log --since=6.months --diff-filter=A --name-only --format= -- <test paths>
   ```

   If new tests follow a different pattern from old ones, the team may be
   migrating. Treat it as an open question, not a rule.
5. **How tests read.** Read three to five recent tests of each kind in full.
   Note:
   - file-name suffix (`.test` or `.spec`, `_test`, `test_`);
   - `describe` and `it` wording, and the language of test names;
   - how the code under test is reached;
   - shared helpers, factories, fixtures, and custom render or setup functions,
     and where they live (`test-utils`, `conftest.py`, `__mocks__/`,
     `fixtures/`, `testdata/`);
   - mocking policy (MSW or module mocks, a real database or a fake, mocked
     clock);
   - assertion style, use of snapshots, table tests, and setup hooks.
6. **Inconsistencies.** Note wherever the repo does the same thing two ways,
   with counts. Do not pick a winner silently.

Done when you can say, for each kind of test, where a new one goes, what it is
called, which helpers it uses, and which command runs it, with evidence, or
that it is unclear.

## 3. Agree with the developer

Show a short draft, one line per convention, each with its evidence and a
status:

- **observed**: consistent, with a count and an example path;
- **mixed**: the repo does it more than one way; show the split;
- **missing**: no tests of this kind yet, or no clear pattern.

Then ask, at most about five questions, using the structured question tool if
the agent has one:

- each **mixed** or **missing** item that affects where or how you will write
  tests;
- whether the team has rules that are not in the repo: review feedback, a wiki
  page, things reviewers keep rejecting. This question matters most, because
  unwritten rules are what get work rejected;
- whether the team is moving away from any pattern you found;
- where to save the record (section 4).

Don't ask about anything the repo already answers consistently. The developer
can correct it in the draft. Don't write or move tests until the developer has
answered, or has said to go ahead with the draft.

## 4. Save the record

Offer two places, and recommend one:

- **Shared**: a `## Test conventions` section in the agent instructions the
  repo already commits (`AGENTS.md` or `CLAUDE.md`), or in the team's existing
  testing doc. Colleagues and their agents see it, and they can correct it in
  review. Recommend it when the repo already keeps agent instructions or a
  testing doc. Leave the change uncommitted for the developer to include in a
  PR; adding rules for the whole team is their call, not yours.
- **This clone only**: `"$(git rev-parse --path-format=absolute --git-common-dir)/kent-test/conventions.md"`.
  Nothing in the repo changes. Recommend it when the repo has no such file, or
  the developer isn't the one who sets team rules.

Write it in this shape, in the language the repo's docs use. Keep it to about
a page; it is read at the start of every session.

```markdown
## Test conventions

<!-- kent-test:conventions -->
Agreed on YYYY-MM-DD at commit <short sha>. Sources: <docs read, counts>.

### Commands
- all: `...`; one file: `...`; CI runs: `...`

### Where tests go
- unit: `src/foo/bar.ts` -> `src/foo/bar.test.ts` (e.g. `src/cart/total.test.ts`)
- integration: `tests/integration/<area>/...`, run by `...`
- e2e: `e2e/<feature>.spec.ts`; new e2e tests only when <rule>

### Naming
- files: ...; `describe` = module name; `it` = "does X when Y", in English

### Helpers, fixtures and mocks
- render with `renderWithProviders` from `test/utils.tsx`; never a bare `render`
- HTTP: MSW handlers in `test/msw/`; no `jest.mock` of the API client

### Don't
- ...

### Exceptions
- `legacy/` keeps its own layout; follow the local pattern there

### Open questions
- ...
```

Every rule gets an example path. Leave out anything you couldn't confirm, or
put it under Open questions.

## 5. Use the record

- **Before writing or moving a test**, check where it goes and what it looks
  like against the record, not only against the nearest sibling file. Make
  sure the runner config and CI will actually pick it up.
- **Precedence.** The record decides where a test lives, what it is called,
  and how it is written. The test-gate value bar decides whether it should
  exist and whether it can fail. When they pull apart, follow the record for
  placement and form, and raise the value question with the developer. Don't
  quietly break the convention. Examples: the gate wants a test at a boundary
  whose test folder the team doesn't use, or the team's usual style matches a
  junk pattern, such as snapshots of rendered copy.
- **Don't restyle the team's tests.** Never rewrite existing tests to suit
  kent-test outside an audit the developer asked for. Even inside an audit,
  moves and merges stay within the record's layout.
- **When the record goes stale.** The repo may contradict it: a new test
  folder, a runner change, a pattern the record never mentions. Point it out,
  follow the repo where it is unambiguous, and propose the edit to the record.
  Don't edit a shared record without asking.
- **Report.** In the final report, name the conventions you followed when
  they decided placement, and any deviation, with its reason.

## 6. No one to ask

If you're running without a developer to ask (a headless or CI run, or a
subagent), don't write a record. Follow the patterns the survey found
consistent and the repo's written rules. Keep tests out of any mixed area
unless the task requires it, and list every convention you assumed, and every
mixed item you met, at the top of the report.
