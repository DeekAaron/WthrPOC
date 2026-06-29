# Technical Context

Engineering-practice contract for code-writing agents in this project. Every implementation must respect what's written here. If a constraint conflicts with the user's request, flag the conflict — don't silently bypass it.

> Stack reconciled: 2026-06-29. Versions below are the floor in use at that date; bump them here when you upgrade.
>
> **Platform scope:** Windows + macOS. Linux was an early aspiration but is **deliberately dropped** — .NET MAUI has no Linux desktop target, and a desktop weather POC does not justify changing UI frameworks to keep it. Revisit (Avalonia/Uno) with an ADR if Linux re-enters scope.

## Overriding Principles
1. **Secrets never reach source and never reach git.** Any provider credential (API key, token) is read at runtime from platform secure storage (`SecureStorage`, backed by the Windows Credential Locker / macOS Keychain) — never hard-coded, never in XAML, never in a committed config. The current provider (Open-Meteo) needs no key; this rule binds the moment a keyed provider is adopted.
2. **Network and OS access live behind an injected service seam, never in views or code-behind.** All outbound calls (the weather provider) and filesystem/OS access sit in a service registered with DI and consumed through an interface (`IWeatherProvider`, etc.). Views and ViewModels depend on the interface, not the concrete client — this is the boundary the tests exercise and the place a provider gets swapped.
3. **No breaking change to a public service contract or a persisted store schema without an ADR.** Those interfaces and on-disk shapes are the app's internal contracts; changing them silently breaks running installs and dependent code.
4. **Cross-platform parity across the supported set (Windows + macOS) is a requirement, not a nice-to-have.** Code that touches the filesystem, paths, or platform APIs must work on both — use `FileSystem`/`Path` abstractions, no OS-only assumptions without a guarded fallback. Linux is out of scope (see header) and need not be accommodated.
5. **Fail visible, not silent.** A failed weather fetch or network error surfaces to the user as a clear state (see User Feedback Approach) and is logged — it is never swallowed into an empty or stale screen.

## Frameworks
- **.NET MAUI (on .NET 9)** — cross-platform desktop application framework; targets Windows (WinUI 3) and macOS (Mac Catalyst).
- **C# 13** — the application language; nullable reference types enabled, warnings-as-errors for nullability.
- **XAML + MVVM** — UI markup with a Model-View-ViewModel separation; Views are thin, logic lives in ViewModels.

## 3rd-party tech
- **Open-Meteo** — weather + forecast data provider. Free, key-less, called from the service layer. Sits behind an `IWeatherProvider` seam so it can be swapped without touching ViewModels or Views.

## Packages in use
**MVVM:** `CommunityToolkit.Mvvm` (source-generated `ObservableObject` / `RelayCommand`)
**HTTP:** `System.Net.Http.HttpClient` via `IHttpClientFactory` (registered in DI; never `new HttpClient()` per call)
**JSON:** `System.Text.Json`
**DI / config / logging:** `Microsoft.Extensions.DependencyInjection`, `.Configuration`, `.Logging` (the MAUI `MauiAppBuilder` host)
**Secure storage:** `Microsoft.Maui.Storage.SecureStorage`
**Testing — unit:** `xUnit`, `NSubstitute` (test doubles at the seams)
**Testing — end-to-end:** `Appium` WebDriver client driving the packaged app per-OS

Reach for these before pulling in a new dependency. A new runtime dependency is a deliberate decision — justify it, don't drift into it.

## Instrumentation
Logging goes through **`Microsoft.Extensions.Logging`**, injected via DI. In dev, logs fan out to the **debug output / console**; in packaged builds, to a **per-platform application log file** under `FileSystem.AppDataDirectory`. Levels: `Error` = a user-visible failure or a broken invariant; `Warning` = a recovered/degraded path (e.g. a provider retry); `Information` = lifecycle and each outbound weather request (resolved location + URL, never credentials); `Debug` = developer detail, off in release. **Always logged:** every outbound provider call and its outcome, every unhandled command/service error, and app start/stop. No metrics or tracing backend for the POC — revisit with an ADR if the app graduates past POC.

## User Feedback Approach
Feedback is **in-app only** — there is no email/Slack channel. Tone is **terse and plain**: state the weather, or state plainly what went wrong. Every async view binds to one of three explicit ViewModel states — **loading**, **result**, or **error** — never a blank or silently-stale screen (per Overriding Principle 5). Error copy names the problem in user terms ("Couldn't reach the weather service — check your connection") and offers the next action (retry), never a raw stack trace or error code. Default to this three-state pattern for any new async view.

## Branching and Environments
- **Model:** Trunk-based. `main` is the single long-lived branch.
- **Branches:** HITL work → a short-lived branch → pull request → `main`. AFK work → one `story/<issue#>-<slug>` branch per Story, owned by the orchestrator (the agent performs no branch operations), squash-merged into `main` on `Approved`. One Story in flight at a time.
- **Environments:** No deployed environments — the artefact is a locally-built desktop bundle (Windows MSIX / macOS app). "Release" = a tagged, signed build produced from `main`.
- **Protected:** `main` — no direct commits; only PRs that pass the `ci` check (and review where required) merge in.

## Git Guardrails
- Never force-push, `git reset --hard`, `git clean`, or delete a branch without explicit confirmation.
- Never commit to `main` directly — all changes flow through a branch and PR.
- Never commit secrets or build output; `.env*`, `dist/`, `build/`, `out/`, `bin/`, `obj/`, and `.orchestrator/` are git-ignored and must stay so.
- CI enforces a **gitleaks** secret scan on every push/PR; GitHub Actions are **pinned by commit SHA** (a tag can be force-moved, a SHA cannot). New build/lint/test steps land behind the existing `ci` job name so the branch-protection rule never needs revisiting.

## Testing & the ratchet
> Seeded from the Factory standard — **opt-out, not opt-in**. Trim a line only with a deliberate, recorded decision; don't start from nothing. Full standard: <https://github.com/kitcox-dev/enate-claude-skills/blob/main/docs/testing-and-the-ratchet-standard.md> · the *why*: <https://github.com/kitcox-dev/enate-claude-skills/blob/main/docs/agentic-e2e-testing-principles.md>.

- **Test the contract, not the LLM.** Assert on the deterministic envelope (output shape, state transition, side-effect ordering), never on generated text or chain-of-thought.
- **Every seam gets a real-IO test on at least one side.** Mock-on-both-sides only proves the mocks agree; defects live at integration seams, not inside modules. For this app the load-bearing seams are the **`IWeatherProvider`** (service ⇄ Open-Meteo) and the **ViewModel ⇄ View binding** — each gets a real-IO test on at least one side.
- **Three tiers, each catching a class the others can't.** Tier 1 — recorded-replay (real local I/O, fakes at the seams, replayed provider-response fixtures; every commit); Tier 2 — live, disposable (real process / real provider call against throwaway state; bounded cost, scheduled); Tier 3 — live (real packaged build, real provider; continuous).
- **Platform matrix is non-negotiable for OS-touching code.** Filesystem/path/platform code and the Appium end-to-end runs execute on **Windows and macOS** — the full supported set, not just the dev box's OS. (Linux is out of scope per the header.)
- **Plan test coverage at the Feature level, not the Story level.** Each Feature declares at kickoff which tiers it needs, which fixtures/substrates must exist, and a Tier-2 cost ceiling.
- **The ratchet — every Tier-3 defect pins a cheap-tier regression.** A live defect's fix lands with a Tier-1 (or Tier-2) regression that would have caught it, encoding the **class** of defect **bounded by the seam** where it surfaced — not the exact reproducer, and not a project-wide sweep. A Story that surfaced from a live defect carries that ratcheting fixture as an acceptance criterion.
- **Disposable substrate, not shared.** Tier-2 substrate is created per run and torn down; reproducibility comes from a seed (fixture/manifest/script), never persistent state.
- **Assert on observable behaviour, not internal state.** For Tier 2/3 the target is what an operator could verify from external artefacts — the rendered state, the logged request/outcome, the on-disk store contents.
