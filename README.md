# Butler

A personal agent for the Mac mini and iPhone. It runs errands in native Mac apps through the accessibility tree, learns new errands by watching them done once, and asks for approval from your iPhone or the Mac menu bar. Most of the time you never open the app.

- [DESIGN.md](DESIGN.md): principles, the Errand component family, motion, the state matrix and the API inventory.
- [docs/DECISIONS.md](docs/DECISIONS.md): open decisions, options and running costs.
- [SECURITY.md](SECURITY.md): threat model and controls.

## Status

Phase 0 (foundation): design contract, decisions and repository security. Next is Phase 2a, a headless runner built with SwiftPM before the UI phases (docs/DECISIONS.md D9). No app code yet.

## Setup

```bash
git config core.hooksPath .githooks
```

Requires gitleaks (`brew install gitleaks`). Building the app requires Xcode 27.
