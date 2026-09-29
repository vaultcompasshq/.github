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
conductor init             # writes .guardrails.yaml; add --hook for one pre-commit hook
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
      - uses: actions/checkout@v7
        with:
          fetch-depth: 0
      - name: Install gitleaks and osv-scanner
        run: |
          set -euo pipefail
          bin="$RUNNER_TEMP/external-gates"
          mkdir -p "$bin"
          cd "$RUNNER_TEMP"
          curl -sSLO https://github.com/gitleaks/gitleaks/releases/download/v8.30.1/gitleaks_8.30.1_linux_x64.tar.gz
          echo "551f6fc83ea457d62a0d98237cbad105af8d557003051f41f3e7ca7b3f2470eb  gitleaks_8.30.1_linux_x64.tar.gz" | sha256sum -c -
          tar -xzf gitleaks_8.30.1_linux_x64.tar.gz -C "$bin" gitleaks
          curl -sSL -o "$bin/osv-scanner" https://github.com/google/osv-scanner/releases/download/v2.6.0/osv-scanner_linux_amd64
          echo "ca69b3d3cd08f889a49dc0a383122f71cc528b83803671df5fd874d97485b108  $bin/osv-scanner" | sha256sum -c -
          chmod +x "$bin/osv-scanner"
          echo "$bin" >> "$GITHUB_PATH"
      - uses: vaultcompasshq/conductor@b0b675a48e7f0e38efc54ec80861b3bafbfcd14f # v0.6.0
      - uses: github/codeql-action/upload-sarif@v4
        if: always()
        with:
          sarif_file: conductor.sarif
```

One pin, not five. The action tag decides which version of each of the
four npm packages is installed, and those defaults are the versions that
tag was tested with, so there is no `version` input to set here; the
install step above is required when the policy enables the secrets-history
and vulnerabilities gates, and it pins gitleaks and osv-scanner by version
and checksum rather than by the action tag. Setting one creates a second
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

Built by Vault & Compass, the team behind Prismfolio (https://prismfolio.io) and Sheetful (https://sheetful.io).

Security reports: security@vaultcompass.io. See each repository's
SECURITY.md.
