# Technical Context

Engineering-practice contract for code-writing agents in this project. Every implementation must respect what's written here. If a constraint conflicts with the user's request, flag the conflict — don't silently bypass it.

> Stack reconciled: 2026-06-29. Versions below are the floor in use at that date; bump them here when you upgrade.

## Overriding Principles
1. **Secrets never reach the renderer and never reach git.** Any provider credential (API key, token) lives only in the Rust core, read from the OS keychain via `tauri-plugin-stronghold` / the platform credential store — never hard-coded, never in a `.env` that ships, never passed to the webview. The current provider (Open-Meteo) needs no key; this rule binds the moment a keyed provider is adopted.
2. **The webview is untrusted; the Rust core is the trust boundary.** All outbound network calls and filesystem access happen in Rust `#[tauri::command]` handlers. The renderer asks the core to do work via IPC — it does not call third-party APIs directly. Keep Tauri's capability allowlist and CSP as tight as the feature needs; widen deliberately, never blanket-enable.
3. **No breaking change to an IPC command signature or a persisted store schema without an ADR.** The IPC boundary and on-disk store are the app's contracts; changing them silently breaks running installs.
4. **Cross-platform parity is a requirement, not a nice-to-have.** Code that touches the filesystem, paths, or the OS must work on Windows, macOS, and Linux. No `/`-hardcoded paths, no platform-only assumptions without a guarded fallback.
5. **Fail visible, not silent.** A failed weather fetch or network error surfaces to the user as a clear state (see User Feedback Approach) and is logged — it is never swallowed into an empty or stale screen.

## Frameworks
- **Tauri v2** — desktop application shell. Rust core process + OS-native webview; produces the cross-platform (Windows/macOS/Linux) bundles.
- **React 19** — renderer UI, written as function components with hooks.
- **TypeScript 5.x** — the renderer language; `strict` mode on, no implicit `any`.
- **Vite 6** — frontend dev server and build tool for the webview bundle.
- **Rust (stable, edition 2021)** — the Tauri core: IPC commands, the weather-provider call, and any OS access.

## 3rd-party tech
- **Open-Meteo** — weather + forecast data provider. Free, key-less, called from the Rust core. The provider behind a `WeatherProvider` seam so it can be swapped without touching the UI.

## Packages in use
**Rust core — HTTP:** `reqwest` (async client)
**Rust core — async runtime:** `tokio`
**Rust core — serialization:** `serde`, `serde_json`
**Rust core — logging:** `tauri-plugin-log`
**Renderer — data fetching/IPC:** `@tauri-apps/api` (the `invoke` bridge)
**Renderer — UI:** `react`, `react-dom`
**Renderer — build:** `vite`, `@vitejs/plugin-react`
**Testing — renderer:** `vitest`, `@testing-library/react`, `@testing-library/jest-dom`
**Testing — Rust:** the built-in `cargo test` harness
**Testing — end-to-end:** `tauri-driver` + a WebDriver client (per-OS live runs)

Reach for these before pulling in a new dependency. A new runtime dependency in either the Rust core or the renderer is a deliberate decision — justify it, don't drift into it.

## Instrumentation
Logging goes through **`tauri-plugin-log`**, which fans out to **stdout in dev** and to the **OS log directory in packaged builds** (`tauri::path::app_log_dir`). Levels: `error` = a user-visible failure or a broken invariant; `warn` = a recovered/degraded path (e.g. provider retry); `info` = lifecycle and each outbound weather request (URL + resolved location, never credentials); `debug` = developer detail, off in release. **Always logged:** every outbound provider call and its outcome, every IPC command error, and app start/stop. No metrics or tracing backend for the POC — revisit with an ADR if the app graduates past POC.

## User Feedback Approach
Feedback is **in-app only** — there is no email/Slack channel. Tone is **terse and plain**: state the weather, or state plainly what went wrong. The renderer always shows one of three explicit states — **loading**, **result**, or **error** — never a blank or silently-stale screen (per Overriding Principle 5). Error copy names the problem in user terms ("Couldn't reach the weather service — check your connection") and offers the next action (retry), never a raw stack trace or error code. Default to this three-state pattern for any new async view.

## Branching and Environments
- **Model:** Trunk-based. `main` is the single long-lived branch.
- **Branches:** HITL work → a short-lived branch → pull request → `main`. AFK work → one `story/<issue#>-<slug>` branch per Story, owned by the orchestrator (the agent performs no branch operations), squash-merged into `main` on `Approved`. One Story in flight at a time.
- **Environments:** No deployed environments — the artefact is a locally-built desktop bundle. "Release" = a tagged, signed build produced from `main`.
- **Protected:** `main` — no direct commits; only PRs that pass the `ci` check (and review where required) merge in.

## Git Guardrails
- Never force-push, `git reset --hard`, `git clean`, or delete a branch without explicit confirmation.
- Never commit to `main` directly — all changes flow through a branch and PR.
- Never commit secrets or build output; `.env*`, `dist/`, `build/`, `out/`, and `.orchestrator/` are git-ignored and must stay so.
- CI enforces a **gitleaks** secret scan on every push/PR; GitHub Actions are **pinned by commit SHA** (a tag can be force-moved, a SHA cannot). New build/lint/test steps land behind the existing `ci` job name so the branch-protection rule never needs revisiting.

## Testing & the ratchet
> Seeded from the Factory standard — **opt-out, not opt-in**. Trim a line only with a deliberate, recorded decision; don't start from nothing. Full standard: <https://github.com/kitcox-dev/enate-claude-skills/blob/main/docs/testing-and-the-ratchet-standard.md> · the *why*: <https://github.com/kitcox-dev/enate-claude-skills/blob/main/docs/agentic-e2e-testing-principles.md>.

- **Test the contract, not the LLM.** Assert on the deterministic envelope (output shape, state transition, side-effect ordering), never on generated text or chain-of-thought.
- **Every seam gets a real-IO test on at least one side.** Mock-on-both-sides only proves the mocks agree; defects live at integration seams, not inside modules. For this app the load-bearing seams are the **IPC boundary** (renderer ⇄ Rust command) and the **`WeatherProvider`** (Rust core ⇄ Open-Meteo) — each gets a real-IO test on at least one side.
- **Three tiers, each catching a class the others can't.** Tier 1 — recorded-replay (real local I/O, fakes at the seams, replayed provider-response fixtures; every commit); Tier 2 — live, disposable (real subprocess / real provider call against throwaway state; bounded cost, scheduled); Tier 3 — live (real packaged build, real provider; continuous).
- **Platform matrix is non-negotiable for OS-touching code.** Filesystem/subprocess/path code and the end-to-end `tauri-driver` runs execute on Windows, macOS, and Linux — not just the dev box's OS.
- **Plan test coverage at the Feature level, not the Story level.** Each Feature declares at kickoff which tiers it needs, which fixtures/substrates must exist, and a Tier-2 cost ceiling.
- **The ratchet — every Tier-3 defect pins a cheap-tier regression.** A live defect's fix lands with a Tier-1 (or Tier-2) regression that would have caught it, encoding the **class** of defect **bounded by the seam** where it surfaced — not the exact reproducer, and not a project-wide sweep. A Story that surfaced from a live defect carries that ratcheting fixture as an acceptance criterion.
- **Disposable substrate, not shared.** Tier-2 substrate is created per run and torn down; reproducibility comes from a seed (fixture/manifest/script), never persistent state.
- **Assert on observable behaviour, not internal state.** For Tier 2/3 the target is what an operator could verify from external artefacts — the rendered state, the logged request/outcome, the on-disk store contents.
