# Butler: rules for coding agents

Read DESIGN.md before any UI work and docs/DECISIONS.md before any architecture work. They override habits.

## Stack

- Swift 6.4, strict concurrency, SwiftUI first. AppKit or UIKit only where SwiftUI cannot do the job, with the reason in a comment.
- Targets: macOS 27 (Mac mini runner, menu bar extra, app) and iOS 27 (app, widgets, Live Activity, controls).
- No third-party UI libraries. Dependencies need a line in DECISIONS.md.

## Slices and phase gates

- Work happens in phases (DESIGN.md section 12). One phase is one or more PRs, each a small reviewable slice.
- A phase ends with proof: Preview Snapshot variants (light/dark, Dynamic Type sizes, Live Activity states, widget timelines) and a screen recording of the flow. Then stop and wait for approval.
- Phase 2a (headless runner, DECISIONS.md D9) has no UI: its proof is a CLI transcript, the receipt JSON with the full state trail, and a screen recording.
- Models and harnesses are planners only (DECISIONS.md D8). Never let one own Errand state, Policy, approvals or receipts.
- Before writing code in a phase, list the Apple APIs it uses with doc links in DESIGN.md section 12 and confirm their signatures in current docs. Never invent an API. If something in the brief does not exist, say so and propose the closest real option.

## Hard rules

- No glass on glass, no custom backgrounds on system bars, no hard-coded corner radii, no easing curves outside `Motion`. SwiftLint enforces these from Phase 1.
- Every user action is an `AppIntent` first; views call intents.
- Destructive actions (send, delete, buy, submit, overwrite) always need explicit approval, at every trust level.
- Receipts are append-only. Agent code never edits or deletes a receipt.
- No secrets in code, logs, commits or PR text. API keys and the APNs key live in the `agentos` keychain. The iPhone holds no secrets.
- Snapshots, clipboard and screenshots can contain private content: never log them raw, never commit them.

## Git

- Branch `claude/<topic>` (or `codex/<topic>`) in its own worktree under `.worktrees/`. Never commit to `main`.
- Enable the secret gate once per clone: `git config core.hooksPath .githooks`.
- PR description: what, why, risk, how it was tested, what was not verified.

## How we build (keeping cost low)

- Plan mode at the start of each phase. The plan lists the APIs, files and the proof to produce.
- Broad searches go to subagents on fast models. Architecture, security and review use the strongest model.
- Read targeted ranges of large files. Pipe build and test output through `tail` or `grep`; show failures only.
- Codex cross-reviews the runner, Policy and redaction PRs (the `cross-review` skill).
- End each slice with a receipt (the `receipt` skill) and start the next phase in a fresh session.
