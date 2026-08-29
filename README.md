# ctf-writeups
Write-ups from Hack The Box CTFs and TryHackMe Hacker Holiday rooms — documenting
not just the flags, but the reasoning, dead ends, and root causes behind each solve.
Most of these come from **TryHackMe's Hacker Holidays 2026** event (The Byte Lotus
Hotel), with more added as I work through rooms, boxes, and CTFs.

## Index
| Writeup | Category | Difficulty |
|---|---|---|
| [The Concierge Knows Too Much](./the-concierge-knows-too-much.md) | AI / Prompt Injection / Social Engineering | Very Easy |
| [Room 404](./room-404.md) | Web / Directory Enumeration | Very Easy |
| [Towel on the Sunbed](./towel-on-the-sunbed.md) | Web Exploitation / Race Condition | Easy |
| [Complimentary](./complimentary.md) | Cloud / AWS / Cognito / IAM Misconfiguration | Easy |
| [Packed Light](./packed-light.md) | Forensics / PCAP Analysis / Cryptography | Easy |
| [Beach Bar](./beach-bar.md) | Boot2Root / Web / YAML Deserialization | Easy |
| [Overheard at Breakfast](./overheard-at-breakfast.md) | OSINT / Social Media / Hashing | Easy |
| [CryptoCabana](./crypto-cabana.md) | Cloud / Azure / Storage / Key Vault | Medium |
| [Do Not Disturb](./do-not-disturb.md) | Boot2Root / Web / NoSQLi / SSTI | Medium |
| [The Hollow Shell](./the-hollow-shell.md) | Web / Zip Slip / Path Traversal | Medium |

## How these are structured
Each writeup follows the same format:
- **Challenge Summary** — what the scenario/app was and what the objective was
- **Steps Taken** — the actual process, including what *didn't* work and why, not
  just the final working path
- **Root Cause** — the underlying vulnerability class behind the flag, explained
  in plain terms
- **Category Note** — how it maps to broader vuln categories (OWASP, cloud
  misconfig, LLM-specific issues, etc.)

Flags are intentionally omitted or redacted from writeups where they'd trivialize
the challenge for others still working through the same room.
