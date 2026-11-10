# Decisions, options and costs

Open decisions are marked **Open**. A decision is closed by editing this file in a PR with the choice and the date.

Prices are from Anthropic's published model table (cached 2026-11-03) and Apple's developer program page. Per-errand model costs are **estimates** until Phase 2 logs real `usage` numbers. Phase 2 adds a cost line to every Receipt, so these get replaced with measured figures.

---

## D1. Blocker: Xcode is not installed and the disk is 95% full (Open)

This Mac has only the Command Line Tools (Swift 6.4) and about 10 GB free. Phase 1 needs Xcode 27 for previews, the Preview Snapshot MCP tool and simulators. Xcode plus one iOS simulator runtime takes roughly 40–50 GB once installed, with more for DerivedData during builds.

| Option | Cost | Tradeoff |
| --- | --- | --- |
| **A. Free up about 60 GB on the internal disk, install Xcode 27 there (recommended if it is reachable)** | $0 | Fastest builds. Needs a cleanup pass (a read-only `computer-health-check` run shows where the space went). |
| B. Install Xcode, simulators and DerivedData on an external APFS SSD | ~US$80–120 for 1 TB | Works well over USB-C or Thunderbolt. The drive has to stay attached. |
| C. Write Phase 1 code without previews | $0 | Breaks the phase gate (no snapshot proof). Not recommended. |

## D2. Model stack and running cost (Open)

**Assumed usage:** 10 errands a day. 30% are new (planned by a model), 70% are replays of learned skills. About 30% of replays hit a UI change and need one recovery call.

**Per-errand profile used for the estimate:** a new errand is about 80K input tokens (60% served from the prompt cache) and 6K output tokens across planning and about 12 steps. A recovery is about 8K in and 1K out.

| Option | What runs where | Est. per new errand | Est. per month |
| --- | --- | --- | --- |
| **1. Lean** | On-device model: triage, redaction, step summaries. Private Cloud Compute (free, 32K context, reasoning): plans for simple errands. Claude Haiku 5.5: step decisions. Claude Sonnet 5.5: plans only when PCC declines or the context is too long. | ~$0.04 | **~$3–6** |
| **2. Balanced (recommended)** | On-device model: triage, redaction, summaries. Claude Sonnet 5.5: planning, step decisions and recovery. Learned skills replay without a model unless the UI diverges. | ~$0.15 | **~$12–18** |
| 3. Max quality | Same as 2, with Claude Opus 5.5 for planning and recovery. | ~$0.30 | **~$25–35** |

Per-token prices behind these numbers (per 1M tokens, input / output): Haiku 5.5 $0.10 / $0.50, Sonnet 5.5 $2 / $10 (cache reads $0.10), Opus 5.5 $4 / $20 (cache reads $0.20).

The biggest cost levers, in order:

1. **Replay before reasoning.** A learned skill replays recorded steps against fresh snapshots at zero model cost. This is also why Teach mode matters for the product.
2. **Compact snapshots.** Send the model a pruned accessibility subtree around the target app, not the whole tree. Raw snapshots can run to tens of thousands of tokens.
3. **Prompt caching.** The system prompt, tool definitions and Policy text form a stable cached prefix.
4. **Free Apple models first.** Triage, redaction and summaries never leave the device; PCC handles simple plans at no cost.

## D3. Fixed costs

| Item | Cost | When it is needed |
| --- | --- | --- |
| Apple Developer Program | US$99/year (local price varies) | Phase 3: CloudKit, APNs, Live Activity push, App Groups across devices and TestFlight all need it. Phases 1–2 run with a free Personal Team on this Mac. |
| CloudKit private database | $0 | Data counts against your own iCloud storage, not a developer bill. |
| APNs | $0 | |
| GitHub | $0 | Secret-scanning CI runs on a Linux runner. There is no macOS CI: macOS runners are expensive, and the Mac mini builds and tests locally. |
| Claude Code for building Butler | Your existing plan | See "How we build" in CLAUDE.md for how the token spend is kept low. |
| Firecrawl / Exa | Research only | Not used by the product at runtime. |

## D4. agent-desktop: CLI child process first, FFI later (proposed)

The brief prefers FFI. The proposal is to start with the CLI:

- **Kill switch:** terminating a child process stops input to the target app immediately. A function call stuck inside an in-process FFI cannot be interrupted the same way. The acceptance bar is a stop within 1 second.
- **Crash isolation:** a crash while walking a broken accessibility tree takes down the child, not the runner, and the runner writes the receipt.
- **Cost:** about 20–50 ms of process start-up per command. That is small next to model latency, and Phase 6 measures it. FFI stays the upgrade path if measurements call for it.

## D5. Runner identity (assumed)

This Mac mini's account is already `agentos`, the dedicated agent user from the brief. The runner runs as a LaunchAgent in this account, and only the runner binary is granted Accessibility. If you would rather keep a separate human account on this Mac and give Butler its own, say so before Phase 2: it changes which apps and files Butler can reach.

## D6. Browser automation (deferred to after Phase 2)

The Phase 2 errand (Numbers to PDF) needs no browser. Playwright's bundled Chromium adds a Node runtime and several hundred MB of disk, and it runs in its own profile without your logged-in sessions. The alternative, driving Safari through the accessibility tree, uses your sessions but is less robust. Decide when the first web errand is chosen.

## D7. Repository visibility (Open)

The repo is **public**. That costs nothing and keeps GitHub's free secret scanning and push protection, but the design, bundle IDs and CloudKit container names are visible. Nothing secret will be committed either way: the pre-commit hook and CI gitleaks block keys, and APNs keys and API keys live only in the `agentos` keychain. On a private personal repo, GitHub's push protection is not available for free, so the local hook and CI would be the only gate.
