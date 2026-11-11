# Decisions, options and costs

Open decisions are marked **Open**. A decision is closed by editing this file in a PR with the choice and the date.

Prices are from Anthropic's published model table (cached 2026-11-03) and Apple's developer program page. Per-errand model costs are **estimates** until Phase 2 logs real `usage` numbers. Phase 2 adds a cost line to every Receipt, so these get replaced with measured figures.

---

## D1. Blocker: Xcode is not installed and the disk is 95% full (Decided 2026-11-10: Yonis installs Xcode 27; Phase 1 starts when it is ready)

This Mac has only the Command Line Tools (Swift 6.4) and about 10 GB free. Phase 1 needs Xcode 27 for previews, the Preview Snapshot MCP tool and simulators. Xcode plus one iOS simulator runtime takes roughly 40–50 GB once installed, with more for DerivedData during builds.

| Option | Cost | Tradeoff |
| --- | --- | --- |
| **A. Free up about 60 GB on the internal disk, install Xcode 27 there (recommended if it is reachable)** | $0 | Fastest builds. Needs a cleanup pass (a read-only `computer-health-check` run shows where the space went). |
| B. Install Xcode, simulators and DerivedData on an external APFS SSD | ~US$80–120 for 1 TB | Works well over USB-C or Thunderbolt. The drive has to stay attached. |
| C. Write Phase 1 code without previews | $0 | Breaks the phase gate (no snapshot proof). Not recommended. |

## D2. Model stack and running cost (Decided 2026-11-10: use the Claude subscription)

**Decision:** Butler bills Claude to Yonis's Claude plan, not a pay-as-you-go API key. Anthropic's help center (article 15036540, read 2026-11-10) says that `claude -p` and the Claude Agent SDK draw from subscription limits when signed in with the plan. Since 2026-10-07, Max, Ultra and Team plans also include monthly API credits for API keys from the linked Claude Console organization.

That allows two routes, chosen in Phase 2 once the plan tier is confirmed:

| Route | How it works | Tradeoff |
| --- | --- | --- |
| **A. `claude -p` as the planner (any paid plan)** | The runner calls Claude Code headless as a child process, with its built-in tools turned off, and gets a structured plan or recovery step back. Butler's own executor does all acting, under Policy. | Shares limits with your coding sessions. Adds a few seconds of start-up per call. Lives outside the `LanguageModelSession`, so it gets no Dynamic Profile switching. |
| **B. Plan API credits via ClaudeForFoundationModels (Max, Ultra or Team)** | A Console API key from your linked organization, stored in the `agentos` keychain, backs a `LanguageModel` inside the session. | Cleanest integration, with prompt caching and profile switching. When the monthly credits run out, calls stop unless you allow overage. |

Either way the free Apple models (on-device and Private Cloud Compute) handle triage, redaction, summaries and simple plans first, and learned skills replay without a model. When Claude's limit is reached, the errand waits in a "Waiting for Claude" state and the morning briefing reports it. It never silently switches to paid billing.

The estimates below show what the same work would cost at API prices. They help size how much of the plan's usage Butler will consume.

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
| Apple Developer Program | US$99/year (local price varies). Not enrolled yet; enroll before Phase 3. | Phase 3: CloudKit, APNs, Live Activity push, App Groups across devices and TestFlight all need it. Phases 1–2 run with a free Personal Team on this Mac. |
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

The Phase 2a errand reads its sources through the planner (D9) and Numbers to PDF needs no browser. Playwright's bundled Chromium adds a Node runtime and several hundred MB of disk, and it runs in its own profile without your logged-in sessions. The alternative, driving Safari through the accessibility tree, uses your sessions but is less robust. Decide when the first web errand is chosen.

## D7. Repository visibility (Decided 2026-11-10: stays public)

The repo is **public**. That costs nothing and keeps GitHub's free secret scanning and push protection, but the design, bundle IDs and CloudKit container names are visible. Nothing secret will be committed either way: the pre-commit hook and CI gitleaks block keys, and APNs keys and API keys live only in the `agentos` keychain. On a private personal repo, GitHub's push protection is not available for free, so the local hook and CI would be the only gate.

## D8. Runtime boundary: harnesses and routers are replaceable planners (Decided 2026-11-10)

**Decision:** anything that runs a model or an agent harness (`claude -p`, ClaudeForFoundationModels, Apple's on-device and PCC models, a router such as HarnessRouter) sits behind one `Planner` protocol in the runner. It proposes plans, step decisions and recoveries. It never owns Errand state, Policy, approvals, the Accessibility grant or receipts; those stay in the runner on the Mac mini, so a planner can be swapped without touching them.

**HarnessRouter: evaluate later.** A one-off tool test on 2026-11-03 (Codex reading the vendor docs in a browser and typing a recommendation into TextEdit) concluded:

- It exposes existing agent harnesses through one API, with sessions, streamed events, files, cancellation and structured failures. That fits the `Planner` slot.
- Its hosted environment runs remote tasks. Driving native Mac apps would need a separate local executor and an authenticated bridge, which was not verified. Butler already has a local executor (D4), so the router would only ever plan.
- The Community Edition is self-hosted Docker. A Linux container cannot reach native Mac apps.
- The vendor benchmark is one synthetic task, five runs per configuration. Its ~475× cost range changes both model and harness; with the model fixed, the range is 1.5–2.1×. It does not establish savings or reliability for Butler.

Caveats: that test kept no evidence (no receipt, file or recording) and used Codex's own accessibility tool and Playwright, not agent-desktop, so it does not de-risk D4 or D6. The vendor claims above were not independently re-checked. Sources: harnessrouter.ai/docs, /docs/environments, /open-source, /benchmarks.

**How it gets re-evaluated:** once Phase 2a works, run the same errand through a `HarnessRouterPlanner` and compare the two receipts (cost, steps, recoveries). Adopt it only if the receipts show a gain.

## D9. Phase order: headless runner (Phase 2a) before Phase 1 (Decided 2026-11-10)

**Decision:** while D1 (Xcode 27 and disk space) is open, build the runner first as a SwiftPM package with the Command Line Tools. They include Swift 6.4 and the macOS 27 SDK with FoundationModels and ApplicationServices, which is all a headless runner needs. Phase 1 (components and previews) follows once Xcode is installed; Phase 2 then adds the menu bar extra and LaunchAgent on top of the same runner.

Phase 2a is three slices, one PR each:

1. **Runner core:** Errand state machine (DESIGN.md §4), Policy, append-only receipts, store, tests.
2. **Executor:** agent-desktop CLI as a child process (D4), snapshot → act → re-snapshot, kill switch, Accessibility trust check.
3. **Planner and first errand:** `Planner` protocol (D8) with `claude -p` as the first implementation (D2 route A). The first errand is "review a vendor's docs and save a recommendation in TextEdit": the planner reads the sources (no browser automation, so D6 stays deferred), the plan is approved at a CLI prompt, the executor writes and saves the document to ~/Inbox, and the Proof step attaches the file hash and sources to the receipt. Numbers to PDF becomes the second errand.

**Proof for Phase 2a** replaces preview snapshots (there is no UI): a CLI transcript, the receipt JSON showing the full state trail, and a screen recording of the run.

**Not covered by 2a:** iPhone approval, menu bar, Live Activity, learn-by-watching and browser automation.
