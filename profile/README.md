# Vault & Compass

Open-source guardrails for AI-assisted development. Four small tools that
run as a pre-commit hook on your machine and as a GitHub Action on every
pull request, and answer three questions about a change before it lands:
what did it add, what did it leak, and was it what you asked for.

| Gate | Question it answers | Install |
| --- | --- | --- |
| [conductor](https://github.com/vaultcompasshq/conductor) | Runs every gate below and writes one SARIF log | [Marketplace](https://github.com/marketplace/actions/conductor-guardrail-gates) · [npm](https://www.npmjs.com/package/@vaultcompass/conductor) |
| [dep-guard](https://github.com/vaultcompasshq/dep-guard) | Is this new dependency a typosquat, a hallucinated name, a tampered lockfile entry, or an install script? | [Marketplace](https://github.com/marketplace/actions/dep-guard-dependency-gate) · [npm](https://www.npmjs.com/package/@vaultcompass/dep-guard) |
| [vault-guard](https://github.com/vaultcompasshq/vault-guard) | Is there a credential in this diff? | [Marketplace](https://github.com/marketplace/actions/vault-guard) · [npm](https://www.npmjs.com/package/@vaultcompass/vault-guard) |
| [intent-guard](https://github.com/vaultcompasshq/intent-guard) | Does this change stay inside the intent contract that was frozen for it? | [Marketplace](https://github.com/marketplace/actions/intent-guard) · [npm](https://www.npmjs.com/package/@vaultcompass/intent-guard) |

## On your machine, one hook for all three gates

```sh
npm install -g @vaultcompass/conductor @vaultcompass/dep-guard @vaultcompass/vault-guard @vaultcompass/intent-guard
conductor init --dry-run   # prints every file it would write, writes nothing
conductor init             # writes .guardrails.yaml and one pre-commit hook
```

The hook runs every enabled gate on each commit and prints one line when the
commit is clean. Each gate can also be installed and run on its own.

## In CI, one workflow for all three gates

```yaml
name: guardrails
on: [pull_request, push]
permissions:
  contents: read
  security-events: write
jobs:
  gates:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: vaultcompasshq/conductor@v0.4.0
      - uses: github/codeql-action/upload-sarif@v4
        if: always()
        with:
          sarif_file: conductor.sarif
```

One pin, not five. The action tag decides which version of each gate is
installed, and those defaults are the versions that tag was tested with,
so there is no `version` input to set here. Setting one creates a second
pin that Dependabot cannot see: it moves the tag and leaves the input
untouched, and the two drift apart silently.

The Action installs those versions outside the repository it is judging.
On a pull request, every gate reads its rules from the base branch, so a
change cannot turn off the check that exists to catch it.
Findings land in the pull request's code scanning tab through SARIF.

## Why these exist

AI coding assistants are fast and confident, and they make three kinds of
mistake that a reviewer skimming a large diff will miss: they add packages
that do not exist or are not the one you meant, they paste credentials into
files that get committed, and they drift from the task you gave them. Each
gate is narrow on purpose, fast enough to sit in a pre-commit hook, works
offline by default, and installs with one command.

## Also from Vault & Compass

[Prismfolio](https://vaultcompass.io/products/prismfolio/) and
[Sheetful](https://vaultcompass.io/products/sheetful/) are our consumer
finance products. The guardrails above are what we built to keep our own
AI-assisted development of them safe, and they are open source under MIT.

Security reports: security@vaultcompass.io. See each repository's
SECURITY.md.
