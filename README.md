# rooket-claude

[![workflow-lint](https://github.com/jhoblitt/rooket-claude/actions/workflows/workflow-lint.yml/badge.svg)](https://github.com/jhoblitt/rooket-claude/actions/workflows/workflow-lint.yml)
[![OpenSSF Scorecard](https://api.scorecard.dev/projects/github.com/jhoblitt/rooket-claude/badge)](https://scorecard.dev/viewer/?uri=github.com/jhoblitt/rooket-claude)

A [Claude Code plugin marketplace](https://code.claude.com/docs/en/plugin-marketplaces)
with one plugin, `rooket`, whose skill drives
[rooket](https://github.com/jhoblitt/rooket): disposable one-worker Rook
clusters on kind, brought up, reached, diagnosed and torn down through rooket
alone.

## Install

Inside Claude Code:

```text
/plugin marketplace add jhoblitt/rooket-claude
/plugin install rooket@rooket-claude
```

## Usage

The `rooket` skill loads for any work that brings up, populates, reaches,
diagnoses or tears down a rooket cluster. It never touches the ambient
kubectl context.

## Development

Commit messages follow [Conventional Commits](https://www.conventionalcommits.org/);
commitlint enforces this on every pull request. Every GitHub Action is pinned
to a commit SHA: run `pinact run` after editing a workflow and `actionlint`
before committing it. Where a `Makefile` is present, `make check` is the local gate,
`make tools` installs the pinned linter, and that pin lives in the `Makefile`.

## License

[Apache-2.0](LICENSE)
