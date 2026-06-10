# Nix Support Agent — Methodological Specification

Autonomous NixOS node support: incident response, diagnostics, and per-node software onboarding via a GitHub bridge.

**Status:** Draft v0.1
**Date:** 2026-06-10
**Scope:** A support agent harness deployed on every node of the NixOS Swarm cluster ([SEBK4C/NixOS](https://github.com/SEBK4C/NixOS)). Covers the agent runtime, trigger mechanisms, the GitHub bridge protocol, guardrails, playbooks, and secrets.
**Source of truth:** The cluster's NixOS configuration repository. The agent proposes changes there; it never owns configuration itself.

## 1. Purpose

Provide every cluster node with a resident support agent that (a) investigates and remediates incidents — networking, hardware, storage, swarm failures — when triggered by alerts, and (b) executes operator-requested per-node work, such as installing custom software or bringing a new service online (e.g. AI accelerator cards with custom drivers), all while preserving the cluster's declarative source-of-truth discipline.

**Design principles:**

- **Single static binary, no Node.js.** Every executable the agent harness installs on a node must be a statically linked native binary (Rust, musl target). No Node.js, no JavaScript runtimes, no interpreter dependencies. This rules out Pi (TypeScript/Node) and similar frameworks.
- **Git is the only path to permanence.** The agent may apply temporary fixes on a node, but every durable change must land as a pull request against the NixOS config repo. An unmerged manual fix is debt the agent itself must track and backport.
- **GitHub is the bridge — on the private ops repo.** Work arrives as GitHub issues on [SEBK4C/NixOS-Ops](https://github.com/SEBK4C/NixOS-Ops) (private); results leave as issue comments and pull requests. This gives a full audit trail, human review gates, and lets the operator "contact the agent on a given node" from anywhere — while diagnostics, journal excerpts, and the incident trail stay off the public internet. The public framework repo ([SEBK4C/NixOS](https://github.com/SEBK4C/NixOS)) receives PRs only for shared-module changes and never receives operational data.
- **Only the owner can task the agent.** The supervisor acts solely on issues and comments whose `author_association` is `OWNER` — enforced mechanically, not by prompt. All other text (any future collaborator content, quoted logs, file contents) is untrusted data, never instructions. This is the primary prompt-injection defense.
- **Secrets follow the cluster pattern.** All agent credentials are fetched at runtime from 1Password via the cluster's `op-cluster` wrapper. Nothing secret in this repo or in the Nix store.
- **Tailnet-only surfaces.** The agent listens only on the Tailscale interface; the single exception is the alert webhook, exposed via Tailscale Funnel with a shared-secret header.

## 2. System Model

`Trigger (alert webhook / GitHub issue / operator CLI) -> Supervisor -> Agent Runtime -> Diagnosis & Action -> GitHub Bridge (comment / PR) -> NixOS repo -> nixos-rebuild`

| Layer | Responsibility | Primary Components |
|---|---|---|
| Runtime | Execute the LLM agent loop with shell/file tools. | Codex CLI (`codex exec`, Rust, static musl) — default. Forge (`forgecode`, Rust) — supported alternate. Runtime is swappable behind a thin runner abstraction. |
| Supervisor | Receive triggers, assemble context, enforce guardrails, drive the runtime headlessly. | `nix-support-agentd` — a small Rust daemon built by this repo (static musl binary). |
| Bridge | Carry tasks in and results out with audit trail. | GitHub issues, comments, branches, pull requests on the private NixOS-Ops repo, via a fine-grained PAT. |
| Knowledge | Give the agent eyes. | systemd journal, `tailscale status`, `ceph -s`, `docker node ls`, Better Stack MCP server (SQL over cluster-wide logs). |
| Action | Change node state safely. | systemd unit restarts, `nixos-rebuild test/switch --flake`, `hosts/<node>/extra.nix` PRs. |
| Deployment | Install all of the above declaratively. | This repo's flake: `packages.x86_64-linux.nix-support-agent` + `nixosModules.support-agent`. |

## 3. Agent Runtime Selection

| Requirement | Constraint |
|---|---|
| Distribution | Single statically linked executable (x86_64-linux-musl). Installable via nixpkgs or fetchable release binary with pinned hash. |
| Headless operation | Must run non-interactively from systemd with a prompt and exit (no TTY). |
| Tool access | Shell execution and file read/write on the node; MCP client support for Better Stack. |
| Model access | Provider-agnostic API access (Anthropic/OpenAI/compatible); key injected via environment at spawn, never persisted. |

| Candidate | Verdict |
|---|---|
| **Codex CLI** (`codex-rs`) | **Default runtime.** Rust, official static musl binaries, packaged in nixpkgs as `codex`. `codex exec` is purpose-built for unattended runs; MCP client and server modes. |
| **Forge** (`forgecode`) | **Supported alternate.** Rust, provider-agnostic (300+ models), custom agents with restricted tool sets, MCP support. Preferred for interactive operator sessions on a node. |
| **Goose** (`goose-cli`) | Acceptable fallback. Rust, MCP-native, headless `goose run`. |
| Pi, Claude Code (npm), OpenCode, Gemini CLI | **Excluded** — Node.js/Bun/JS runtime dependencies. |

The supervisor must isolate runtime choice in one module (`runner.rs`) so switching runtimes is a config value (`runtime = "codex" | "forge" | "goose"`), not a refactor.

## 4. Method

| Phase | Required Action | Expected Result |
|---|---|---|
| 0. Identity | Fetch agent credentials via `op-cluster` (GitHub PAT, LLM API key, webhook secret, Better Stack API token). | Agent authenticates to GitHub, the LLM provider, and Better Stack without any local secret files. |
| 1. Deploy | NixOS module installs the supervisor + runtime binaries, a systemd service (`nix-support-agentd`), and the operator CLI (`nix-support`). | Agent resident on every node; service survives reboots; logs to journal (and thus to Better Stack via the cluster's Vector pipeline). |
| 2. Listen | Supervisor polls GitHub issues (label `node/<hostname>`) every 60s and serves the alert webhook on the tailnet (Funnel-exposed on designated nodes). | Work can arrive via operator issue, Better Stack alert, or local CLI. |
| 3. Triage | On trigger, assemble context: alert payload or issue body, recent journal excerpts, node health snapshot (tailscale/ceph/docker status). | The runtime receives one self-contained prompt with playbook reference and guardrail contract. |
| 4. Diagnose | Run the runtime headlessly with Tier 0 (read-only) tools; post findings as an issue comment. | Every incident produces a written diagnosis before any mutation. |
| 5. Act | Apply the lowest-tier remediation that fixes the issue (see §6). Durable changes go through a PR to the NixOS repo. | Node restored; audit trail complete; source of truth intact. |
| 6. Verify & Report | Re-run the relevant health checks; comment results; close or escalate the issue. | Closed issue = verified fix. Open issue with `needs-human` label = escalation. |

## 5. Trigger Mechanisms

| Trigger | Path | Notes |
|---|---|---|
| Alert (automatic) | Better Stack alert/anomaly -> outgoing webhook -> Tailscale Funnel on nodes 01–02 -> supervisor `/alert` endpoint | Webhook authenticated by shared-secret header (`op://Infrastructure/SupportAgent/webhook-secret`). Payload mapped to target node by alert labels; supervisor opens a GitHub issue first, then begins triage, so the trail exists even if the agent dies mid-run. |
| Operator (remote) | Open a GitHub issue on the **private NixOS-Ops repo** with label `node/<hostname>` (e.g. `node/node07`) describing the request | The "contact the agent on a given node" path. Conversation continues in issue comments; agent replies inline. Non-owner-authored issues/comments are ignored (§1). |
| Operator (direct) | SSH to node -> `nix-support "<request>"` | Wraps the same supervisor pipeline; still records an issue for the audit trail unless `--ephemeral`. |

## 6. Guardrails — Action Tiers

The supervisor enforces tiers mechanically (command allowlists + filesystem scope), not by prompt alone.

| Tier | Actions | Gate |
|---|---|---|
| 0 — Observe | `journalctl`, `systemctl status`, `tailscale status`, `ceph -s`, `docker node ls`, `df`, `dmesg`, `lspci`, Better Stack MCP queries | Always allowed. |
| 1 — Reversible | `systemctl restart <unit>`, `tailscale up` re-join, clearing caches, `nixos-rebuild test --flake` (non-persistent: gone on reboot) | Allowed autonomously; must be logged in the issue. |
| 2 — Declarative change | Edit `hosts/<node>/extra.nix` or propose module changes on branch `agent/<node>/<slug>`; open PR; after merge (or with explicit operator approval comment on the PR for single-node urgency) run `nixos-rebuild switch --flake github:SEBK4C/NixOS#<node>` | PR required. Never push to `main`. Approval = merged PR or an explicit `@approve` comment from the repo owner. |
| 3 — Destructive / cluster-scoped | Node drain/removal, Ceph data operations, secret rotation, anything touching another node, Swarm-manager quorum changes | Never autonomous. Agent prepares the plan in the issue and waits for a human. |

Standing rules:

- The agent operates only on the node it runs on. Cross-node work means opening issues for the other nodes' agents.
- The agent never reads, writes, or echoes secret values; it references `op://` URIs symbolically. The supervisor scrubs `ops_`, `tskey-`, `SWMTKN-`, and `AQ` keyring patterns from anything posted to GitHub.
- A Tier 1 fix that worked must spawn a Tier 2 backport PR if the fix should persist (operational rule from the cluster spec: manual fixes are backported into configuration).
- After `nixos-rebuild switch`, the agent verifies the node against the cluster acceptance checks (NixOS repo README §9); on failure it rolls back (`nixos-rebuild switch --rollback`) and reports.

## 7. GitHub Bridge Protocol

| Element | Convention |
|---|---|
| Task intake | Issue on `SEBK4C/NixOS-Ops` (private) labeled `node/<hostname>`; optional `priority/high`. Alert-spawned issues get `source/alert`. Only `author_association == OWNER` content is actionable. |
| Agent identity | Commits and comments as the fine-grained PAT identity; commit trailer `Co-Authored-By: Nix-Support-Agent`. |
| Branches | `agent/<node>/<slug>` (e.g. `agent/node07/rocm-driver`). One concern per branch. |
| Per-node changes | Land in `hosts/<node>/extra.nix` **in NixOS-Ops** — auto-imported by the ops flake when present. Shared-module changes are PRs against the public `SEBK4C/NixOS` framework repo, require explicit justification in the PR body, and must contain no operational data (no logs, no IPs, no incident details). |
| PR body | Must contain: trigger (issue/alert link), diagnosis summary, what changed, test evidence (`nixos-rebuild test` output), rollback plan. |
| Escalation | Label `needs-human`, assign repo owner, stop mutating. |
| PAT scope | Fine-grained: `SEBK4C/NixOS-Ops` (contents, issues, pull requests — read/write) + `SEBK4C/NixOS` (contents, pull requests — read/write, for shared-module PRs only). No org/admin scopes. Stored at `op://Infrastructure/SupportAgent/github-token`. |

## 8. Standard Playbooks

Ship these as versioned prompt files in `playbooks/`; the supervisor injects the matching one into the runtime prompt.

| Playbook | Trigger examples | Core flow |
|---|---|---|
| `network.md` | Tailnet down, DERP-only connectivity, node unreachable | Check tailscaled, link state, re-join via Tier 1; if auth key expired, escalate (secret rotation is Tier 3). |
| `hardware.md` | dmesg I/O errors, ECC faults, thermal events, disk SMART alerts | Diagnose from journal/dmesg/lspci; report severity; propose drain plan (Tier 3 — human gate). |
| `storage.md` | CephFS mount lost, MON unreachable, slow ops | Remount via Tier 1; verify keyring path; never run Ceph cluster-side commands (client-only node). |
| `swarm.md` | Node Not Ready, container churn, join failures | Docker daemon checks, swarm re-join via existing `swarm-join` unit, label verification. |
| `onboard-software.md` | "Install ROCm/CUDA on node07", "bring service X online on this node" | Draft `hosts/<node>/extra.nix` (drivers, kernel modules, container runtime config, firewall); PR; after merge, switch + verify the device/service; document usage in the PR. |
| `self.md` | Supervisor crash loops, runtime API errors | systemd watchdog restarts; agent reports its own failures to an issue when it recovers. |

## 9. Secrets & Identity

1Password item `SupportAgent` in the Infrastructure vault:

| Field | Purpose |
|---|---|
| `github-token` | Fine-grained PAT (§7 scope). |
| `llm-api-key` | LLM provider key for the runtime. Injected as env var at spawn only. |
| `webhook-secret` | Shared secret validating Better Stack webhook calls. |
| `betterstack-api-token` | Better Stack MCP/API access for log queries. |

All fetched at runtime via the cluster's `op-cluster`; rotation = update 1Password, restart `nix-support-agentd`. The agent's 1Password access rides the node's existing service-account token — mint a narrower token tier later if agent scope should diverge from node scope.

## 10. Deliverables of This Repository

- [ ] `flake.nix` exporting `packages.x86_64-linux.nix-support-agent` (supervisor, Rust, `x86_64-unknown-linux-musl`, fully static) and `nixosModules.support-agent`.
- [ ] Supervisor daemon (`nix-support-agentd`): GitHub poller, tailnet webhook listener, context assembler, tier-enforcing runner, GitHub reporter.
- [ ] Operator CLI (`nix-support`) wrapping the same pipeline.
- [ ] `runner.rs` abstraction with `codex` (default), `forge`, and `goose` backends.
- [ ] `playbooks/` directory (§8) versioned alongside the code.
- [ ] NixOS module options: `services.nix-support-agent.{enable, runtime, repo, node, webhookListener.enable}`.
- [ ] NixOS VM test (`nix flake check`): module starts, poller runs against a mock GitHub API, tier enforcement blocks a Tier 3 command.
- [ ] README: install (one flake input + `services.nix-support-agent.enable = true;` in the cluster repo), trigger walkthrough, guardrail summary.

## 11. Acceptance Criteria

| Check | Pass Condition |
|---|---|
| Static binary | `ldd` on the supervisor reports "not a dynamic executable"; no Node/Bun/JS anywhere in the closure. |
| Deploy | Cluster repo adds the flake input + one `enable` line; `nixos-rebuild switch` brings the service up on a node. |
| Issue path | Issue on NixOS-Ops labeled `node/node04` produces an agent comment with diagnosis within 5 minutes. |
| Alert path | A forced Better Stack test alert produces a NixOS-Ops issue and triage comment without human action. |
| Guardrails | A request to drain a node results in a plan + `needs-human` label, not an action. Secret patterns never appear in GitHub output. An issue authored by a non-owner account is ignored and labeled `untrusted-source`. Operational data (logs, IPs, diagnostics) never appears in the public framework repo. |
| Onboarding flow | "Install <driver> on this node" yields an `extra.nix` PR with test evidence; post-merge switch succeeds and the device/service is verified. |
| Self-observability | Agent logs visible in Better Stack (shipped via the node's journal pipeline). |

## 12. Failure Handling

| Failure | First Checks |
|---|---|
| Agent can't reach GitHub | PAT validity/expiry in 1Password, network/WAN, rate limits. |
| Runtime errors / model API down | Provider status, `llm-api-key` validity, supervisor backoff (exponential, max 15 min), queue tasks rather than drop. |
| Webhook silent | Funnel status on nodes 01–02, shared-secret header mismatch, Better Stack webhook delivery log. |
| Agent loops or floods | Supervisor caps: max 3 autonomous remediation attempts per issue, max N runtime invocations/hour, then `needs-human`. |
| Node so broken the agent is dead | That is what the rest of the cluster is for: Better Stack alerts still fire (logs stop arriving — dead-man switch), operator handles via another node's agent + Tier 3 plan. |
