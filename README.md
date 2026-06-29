# Weather POC

A proof-of-concept product built with the **Enate SDLC Factory**.

This is the human front door. The *agent* front door is [`CLAUDE.md`](./CLAUDE.md), which
Claude Code auto-loads every session.

## Status

Just bootstrapped — the repo mechanism is stamped (agent orientation, branching rules, and a
stable CI gate), but no stack or features exist yet. The engineering contract and domain
glossary come next:

1. `Technical-Context.MD` — the engineering contract (run `/init-tech-context`).
2. `business-domain-context.md` — the domain glossary (run `/init-context`).

## Contributing

See [`CONTRIBUTING.md`](./CONTRIBUTING.md) for the branching & merge rules. In short: `main`
only moves through a pull request that passes CI — no direct commits.
