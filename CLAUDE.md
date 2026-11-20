# promptshape

Python library (with a small Rust extension) that renders structured data into prompt text
(XML tags, Markdown, JSON, plain text) and measures the token cost of each format.

Read first: `docs/HANDOFF.md` (state, next task), then that task's entry in `docs/SPEC.md` §12.
Reasons for design choices are in `docs/DECISIONS.md`. Do not re-decide them; add a line if you decide something new.

## Stack

- Python 3.10+, managed with **uv**. Lint and format: **ruff**. Types: **pyright** (strict). Tests: **pytest** + **hypothesis**.
- Rust: **cargo** workspace, **PyO3** bindings built by **maturin** (mixed project, abi3-py310), benches with **criterion**.
- Layout: `python/promptshape/` (package), `rust/bpe/` (pure Rust core), `rust/py/` (PyO3 module `promptshape._bpe`),
  `tests/`, `benches/`, `scripts/`, `examples/`, `docs/`.

## Commands

Run from the repository root. T01 creates the scaffold; if a command below does not match it, fix this file in the same PR.

```bash
uv sync --all-extras --dev                 # install everything, builds the Rust extension
uv run maturin develop --uv                # rebuild the extension after Rust changes
uv run ruff check .                        # lint
uv run ruff format --check .               # format check (drop --check to format)
uv run pyright                             # strict type check
uv run pytest -q                           # all Python tests
uv run pytest tests/test_xml.py -q -k name # one file / one test
cargo fmt --all --check                    # Rust format check
cargo clippy --workspace --all-targets -- -D warnings
cargo test -p promptshape-bpe              # Rust unit tests (no Python needed)
cargo bench -p promptshape-bpe             # criterion, local only, never in CI
uv run python benches/compare.py           # vs tiktoken and tokenizers, local only
```

Human only (needs a key, never run by an agent or CI):

```bash
uv run python scripts/live_count.py --live
```

## Conventions

- Tests first. Write the failing test from the task's "Tests" line, then the code. Never weaken or delete a test to get green.
- Public API is what `promptshape/__init__.py` exports; everything else lives in `_private` modules. Fully typed, no `Any` in public signatures.
- Callers never see format details: no public escaping helpers, tag names or separators.
- Data types are frozen dataclasses. Output is deterministic and byte-exact; goldens in `tests/golden/` are compared as bytes.
- Never use `str.splitlines()` in renderers or test parsers (it splits on U+2028 and friends). Split on `"\n"` only.
- Library code never reads environment variables and never prints. `os.environ` is allowed only in `cli.py`, `scripts/` and `examples/`.
- Errors derive from `PromptshapeError`. Invalid input raises; it is never silently repaired, except where SPEC §5 says so.
- ruff line length 100. Docstrings on public names only. Comments explain why, not what.
- Every PR updates `CHANGELOG.md` (Unreleased, keep-a-changelog) when behaviour changes, and `docs/HANDOFF.md` always.

## Hard rules

- **API key.** The library never reads `ANTHROPIC_API_KEY`. Do not set, export, echo or commit it, and do not add it to
  `.env`, CI or examples. With it in the environment, Claude Code bills the API account instead of the subscription.
  The CLI, `examples/` and `scripts/` read `PROMPTSHAPE_ANTHROPIC_API_KEY` only.
- **No live calls in tests.** Tests use `tests/fakes.py`. The autouse fixtures in `tests/conftest.py` fail any test that
  builds a real Anthropic client or opens a socket. Never add an opt-out. Live calls exist only in `scripts/live_count.py`.
- **Human-authored code.** `train_merges` and the encode merge loop are written by the human author. Never write, complete
  or edit code between `// BEGIN HUMAN-AUTHORED` and `// END HUMAN-AUTHORED`. You may write tests and callers around it.
- **Releases.** Never create tags, GitHub releases or PyPI uploads. The human approves the `testpypi` and `pypi` environments.
- **Settings.** Do not change `.claude/settings.json` or `.claude/agents/`. The sandbox denies reads of `~/.ssh`, `~/.aws`,
  `~/.zshrc`, `~/Documents`, `~/Downloads` and `~/Desktop` outside this repository. Do not work around a denial; report it.

## Branch and PR workflow

1. One task = one branch = one PR. Branch from up-to-date `main` as `tNN-short-slug` (for example `t05-xml-renderer`).
2. Touch only the files the task lists. If the task needs a file it does not list, say so in the PR description.
3. Commit messages: `TNN: what changed`. Small commits are fine.
4. Before opening the PR: all commands above pass (use the test-runner agent), then run the reviewer agent and fix its findings.
5. Open the PR with `gh pr create`, title `TNN: task title`, body: what changed, how it was tested, open questions.
6. Never push to `main`. Never force-push (no `--force`, no `--force-with-lease`); to update a branch, add commits or merge
   `main` into it. Never merge a PR; the human merges.
7. CI runs on pull requests only. Docs-only PRs skip the heavy jobs.

## Helper agents (`.claude/agents/`)

| Agent | Model | Use it for | Do not use it for |
|---|---|---|---|
| `Explore` | haiku, read-only | Finding where something lives in this repository. Returns paths and short excerpts. | Design questions, anything outside the repo |
| `test-runner` | haiku | Running one command (pytest, ruff, pyright, cargo, benches). Reports failures only. | Fixing failures, choosing what to run |
| `reviewer` | opus, read-only | Reviewing the branch diff against the task's section in `docs/SPEC.md`, before opening a PR. | Writing code, reviewing its own fixes without a re-run |

Give each agent one narrow instruction. Read the code yourself when a task touches a handful of known files.

## Compact instructions

When compacting this conversation, keep:

- The task ID and title, the branch name, and whether a PR is open (with its URL).
- Files created or changed in this session, and anything left half-done.
- Failing tests or commands: the exact command and the exact error lines.
- Decisions made in this session and why, and any open question for the human.
- Which commands in "Commands" last passed.

Drop: file contents already written to disk, passing test output, exploration that led nowhere, and repeated copies of
SPEC text (re-read `docs/SPEC.md` instead). After compacting, re-read `docs/HANDOFF.md` and the current task in SPEC §12.
