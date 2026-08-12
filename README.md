# CTF-writeups
Write-ups from CTFs, TryHackMe rooms, and hands-on security challenges — documenting
not just the flags, but the reasoning, dead ends, and root causes behind each solve.

Most of these come from **TryHackMe's Hacker Holidays 2026** event (The Byte Lotus
Hotel) and **HTB Cyber Apocalypse 2026 "The Salt Crown"**, with more added as I work
through rooms, boxes, and CTFs.

## Index
| Writeup | Category | Platform | Difficulty |
|---|---|---|---|
| [The Concierge Knows Too Much](./the-concierge-knows-too-much.md) | AI / Prompt Injection / Social Engineering | TryHackMe — Hacker Holidays 2026 | Very Easy |
| [Room 404](./room-404.md) | Web / Directory Enumeration | TryHackMe — Hacker Holidays 2026 | Very Easy |
| [Towel on the Sunbed](./towel-on-the-sunbed.md) | Web Exploitation / Race Condition | TryHackMe — Hacker Holidays 2026 | Easy |
| [Complimentary](./complimentary.md) | Cloud / AWS / Cognito / IAM Misconfiguration | TryHackMe — Hacker Holidays 2026 | Easy |

## How these are structured
Each writeup follows the same format:
- **Challenge Summary** — what the scenario was and what the objective was
- **Steps Taken** — the actual process, including what didn't work and why, not
  just the final working path
- **Root Cause** — the underlying vulnerability class behind the flag, explained
  in plain terms
- **Category Note** — how it maps to broader vuln categories (OWASP, cloud
  misconfig, LLM-specific issues, etc.)

Flags are intentionally omitted or redacted from writeups where they'd trivialize
the challenge for others still working through the same room.

## About

I'm a Cybersecurity Engineering & Technology student building toward research and
security roles internationally. This repo is where I track hands-on practice across
web, cloud, and emerging AI/LLM security — alongside CTF placements, TryHackMe/HTB
work, and other public profiles linked below.

- TryHackMe / HackTheBox / PortSwigger / OverTheWire / CyberDefenders / Root Me
- HackerOne / Bugcrowd / Hacker101
