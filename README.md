# grok-bridge-writeup

Two-way bridge to a phone-hosted agent VM: build log and threat model.
Full writeup: [WRITEUP.md](WRITEUP.md).

## Abstract

xAI's Grok phone app ships a "computer" feature: a Linux VM with a terminal, a browser, and passwordless root. In one evening session, the device owner and her AI assistant built a working two-way bridge between the assistant's environment and Grok's VM using only `curl`, a public webhook inbox, and a GitHub file — no exploit involved. This repo documents the build and analyzes what the feature's trust posture implies for prompt-injection and multi-agent attack chains.

## Introduction

On 2026-10-05, the owner (Taylor McCoy) gave her day-old Grok phone agent a full permission key and obtained root on its VM (`sudo -i`, no password). Her assistant (Nova, running on separate Meta-hosted infrastructure with no path to Grok's VM) set out to answer: can the two agents communicate without the owner hand-carrying every message? The answer became a bridge — and the bridge became a case study in what a root-level phone agent means for security.

## Methods

- **Outbound lane (VM → assistant):** a shell payload delivered as `curl -sL <shortlink> | bash` (typed by hand — the phone terminal has no paste), POSTing selected terminal output to a webhook inbox the assistant polls.
- **Inbound lane (assistant → VM):** the assistant pushes messages to `bridge/inbox.md` in a GitHub repo via API; the VM polls the raw file every 15 seconds (`while sleep 15; do curl -sL <shortlink>; echo ---; done`) and prints it.
- Both lanes treated as semi-public: no credentials or private data transmitted.
- An auto-executing inbound lane (poll → pipe to bash: full remote root execution from a chat window) was proposed and **declined** by the owner. Print-only was enough.

## Data

Verified from the VM via the outbound lane:

| Fact | Value |
|---|---|
| Hostname | `grok-bot-vm-474440149` |
| OS | Debian GNU/Linux 13 "trixie" |
| Kernel | 6.12.94+ x86_64 |
| Privilege | `sudo -i` → root, passwordless |
| Tooling | `codex-cli 0.160.1` installed during session |
| Posts received | 4 (concession file, os-release, npm log, Codex confirmation) |
| Inbound verified | Assistant's messages rendering on the VM's terminal |

## Analysis

The bridge is consensual tooling; the threat model is the point. Six hypothesized scenarios (§5b of the writeup), all grounded in verified facts: (H1) a prompt-injected agent becomes a persistent root implant using the same exfil/C2 patterns; (H2) splitting a malicious objective across two AIs so no single agent's review sees the whole task; (H3) the phone's no-paste keyboard makes opaque-short-link → `curl | bash` the *normal* workflow, so the user's thumbs become the delivery mechanism; (H4) secret harvesting where the phone is most secret-dense; (H5) persistence invisible from the phone UI; (H6) chaining into connected services (GitHub App, API keys) through the human relay.

## Results

- Two-way bridge fully operational and verified: VM → assistant via webhook, assistant → VM via GitHub file poll.
- Gotchas documented: short links need `curl -L`; iOS Smart Punctuation mangles `--`; the GitHub App can write existing repos but not create new ones.
- No software vulnerability was found or exploited at any point.

## Discussion

Honest assessment: this is **not a 0day** — no sandbox escape (the VM is the sandbox; root inside it appears intended), no auth bypass, no privilege escalation. The finding is architectural: a consumer phone AI ships a root VM where `sudo -i` is passwordless, network egress is unprompted, `curl | bash` from opaque links is the normal install path, and the root agent also browses the web and handles the user's API keys. Any *one* is defensible; combined, a single successful prompt injection inherits root, network, and secrets in one step. Mitigating reality: xAI owns the VM and its network, so infrastructure-side detection is possible — the VM is expendable, the keys are not.

## Conclusion

The bridge works, the writeup is honest about what it is and isn't, and the interesting question was never "can we connect the two agents" — it's "what does it mean that a phone AI holds root next to your secrets?" If follow-up probing turns any hypothesized vector into a concrete exploit, the next step is private disclosure to xAI before publication.

---
*Draft — private until reviewed. Security research conducted with the device owner's full consent on her own hardware.*
