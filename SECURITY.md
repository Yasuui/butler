# Security

## Reporting a vulnerability

Report it privately through a [GitHub security advisory](https://github.com/Yasuui/butler/security/advisories/new). Please do not open a public issue.

## Threat model in one paragraph

Butler holds the Accessibility permission on a Mac and can act inside any app it is allowed to touch. The main risks are acting where it should not, leaking what it sees, and being steered by text it reads on screen (prompt injection). The controls below are built into the design (DESIGN.md) rather than added afterwards.

## Runtime controls

- **Least privilege:** only the runner process holds Accessibility, and it runs as the `agentos` user.
- **Policy:** an allowlist of apps and domains Butler may touch, and a never-touch list (Passwords, Keychain Access, banking apps) checked before every action.
- **Action classes:** read, write, destructive. Destructive actions always need explicit approval, at every trust level.
- **Screen text is data:** text read from apps and pages is never treated as an instruction to Butler.
- **Redaction:** the on-device model classifies snapshot and clipboard text before anything reaches a cloud model or a log. Sensitive text never leaves the Mac.
- **Kill switch:** Control Center, the menu bar, Siri and the Lock Screen each stop a run within 1 second and leave a receipt.
- **Audit:** receipts and the audit log are append-only; agent code cannot edit them.
- **No secrets on the phone:** the iPhone only signs intents (approve, deny, ask).

## Repository controls

- `.githooks/pre-commit` blocks signing keys, provisioning files and `.env` files, then runs gitleaks on staged changes.
- CI runs gitleaks on every push and pull request with `contents: read`.
- Every GitHub Action is pinned to a full commit SHA; Dependabot proposes updates.
- CODEOWNERS requires the owner's review.
