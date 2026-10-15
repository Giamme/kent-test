# kent-test

Two agent skills that keep AI-written tests honest.

AI coding assistants are quick to write tests and slow to ask whether a test
should exist. The usual results are a test for every function touched,
assertions that restate the implementation, mocks that hand back the very
value being checked, and regression tests that were never seen failing. These
skills put a check in front of that, and give you a safe way to clean up a
suite that already has the problem.

| Skill | When it runs | What it does |
|---|---|---|
| **test-gate** | Automatically, whenever the assistant writes, changes, or reviews tests | Decides whether a test should exist, where it belongs, and makes regression tests prove they fail on the old code |
| **test-audit** | When you ask for an audit, cleanup, or pruning of existing tests | Finds tests that cost more than they protect, and repairs or removes them. Every removal is backed by breaking the code on purpose and showing another test still catches it |

Both skills are plain `SKILL.md` folders, so any assistant that reads Agent
Skills can use them. They work in any language or repo: instead of assuming a
test runner, they tell the assistant to find the repo's own test commands.

## What the skills check

**test-gate** asks four questions before a test lands:

1. What behavior does it protect?
2. What plausible regression makes it fail?
3. Why doesn't existing coverage already catch that? Search every test file, not just the sibling one.
4. Does it need an export, flag, or hook that production doesn't need? If so, test through the real entry point instead.

It then checks the test against a list of junk patterns, such as:

- no assertion;
- an expected value computed by the code under test;
- a copied list of constants;
- source-text greps;
- a mock that supplies the answer;
- a negative case that fails for the wrong reason;
- a name that promises more than the input exercises.

"Nothing to add" is a valid outcome.

**test-audit** keeps a ledger for every test in scope. Each test gets a
verdict (keep, fix, merge, or drop), a note on what it actually catches, and
the test that keeps covering the behavior if it goes. Its rules:

- **Drops and merges need proof.** Before one is accepted, the production code
  is broken on purpose and a remaining test has to fail.
- **Repairs count as much as removals.** A test that claims more than it checks
  gets a fix verdict, with the exact change.
- **It knows when a test only looks like junk.** A test that restates a list
  can still be the only proof of a user-visible order. The retention rules keep
  such tests.
- **Report-only audits change nothing.** Their experiments run in a throwaway
  git worktree.
- **It scales to a whole subsystem.** A campaign mode
  (`skills/test-audit/references/campaign.md`) covers pruning one subsystem's
  entire test surface.

## Install

Keep the two skills side by side: test-audit reads its value bar from
`../test-gate/SKILL.md`. It still works alone, using a short built-in
fallback. Every method below installs both.

### Claude Code

As a plugin:

```
/plugin marketplace add Giamme/kent-test
/plugin install kent-test@kent-test
```

From a shell: `claude plugin marketplace add Giamme/kent-test && claude plugin install kent-test@kent-test`.

The skills then appear as `kent-test:test-gate` and `kent-test:test-audit`.

### Codex

Codex reads the same plugin:

```sh
codex plugin marketplace add Giamme/kent-test
codex plugin add kent-test@kent-test
```

### GitHub Copilot CLI

```sh
copilot plugin marketplace add Giamme/kent-test
copilot plugin install kent-test@kent-test
```

### Cursor, Gemini CLI, Antigravity, OpenCode, OpenClaude, Windsurf/Devin, Amp, Kiro and others

Install the skills with [`npx skills`](https://github.com/vercel-labs/skills),
which knows where each agent looks:

```sh
npx skills add Giamme/kent-test            # pick agents interactively
npx skills add Giamme/kent-test -g -a cursor -a gemini-cli -a opencode -y
```

### From a clone

To track `main` and update every agent with a `git pull`, clone the repo and
symlink both skills into each agent that's installed:

```sh
git clone https://github.com/Giamme/kent-test ~/src/kent-test
~/src/kent-test/scripts/install --dry-run   # see what would be linked
~/src/kent-test/scripts/install             # every detected agent; or --client claude --client codex
~/src/kent-test/scripts/install --uninstall
```

Install the skills in one place per agent. With the Claude Code plugin
installed, don't also link them into `~/.claude/skills`, or Claude Code lists
each skill twice; the script skips Claude Code when it finds the plugin.

## What it runs

The plugin is instructions only: two `SKILL.md` files and their references.
It has no hooks, scripts, dependencies or network access. The skills have your
agent run the repo's own test commands. During an audit, the agent breaks
production code on purpose to prove a test catches it, then restores it.
Report-only audits run those experiments in a throwaway git worktree.

## How well it works

We evaluated both skills on a private production codebase (a Rust and
TypeScript desktop app) with Claude Opus 5.5. Each task ran once with the
skill and once without it, and every result was re-verified by hand. The
evaluation fixtures aren't included, because they are built from that private
code.

- **test-gate**, 4 tasks: a regression test for a real bug, a "coverage"
  request for already-covered code, a review of a test file with 6 planted
  problems, and Rust validator tests.
  - The baseline assistant was already good: 22 of 23 checks passed without
    the skill, 23 of 23 with it.
  - The skill's main effect was scope. It wrote fewer, tighter tests (17 vs 96
    lines for the regression test; 3 assertions added to an existing test
    instead of a new file with 8 tests) for about 18% more tokens.
- **test-audit**, 3 tasks: a cleanup of a seeded test file, pruning a healthy
  Rust suite, and a report-only audit.
  - With the skill: 14 of 14 checks. Without it: 12 of 14.
  - Without the skill, the assistant merged away the only test proving the
    board's column order. Deliberately reordering the columns then passed
    every test.
  - With the skill, the assistant kept that test, fixed tests whose names
    claimed checks they didn't make, and proved every proposed removal.
  - It costs about 40% more tokens, and a large report-only audit took about
    24 minutes.

In short, a strong model already writes reasonable tests. These skills make
it narrower when adding tests and much safer when removing them. We haven't
measured smaller models yet.

## Credits

Adapted from the `test-audit` skill in [OpenClaw](https://github.com/openclaw/openclaw)
(`.agents/skills/test-audit`), © 2026 OpenClaw Foundation, MIT License. This
version:
- splits the authoring gate from the audit;
- removes the repo-specific tooling;
- adds per-candidate mutation proof, repair-first verdicts, and report-only
  isolation.

## License

MIT. See [LICENSE](LICENSE). OpenClaw's notice ships in each skill folder as
`LICENSE-openclaw`.
