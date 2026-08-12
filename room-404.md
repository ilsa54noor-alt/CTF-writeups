# Room 404 — TryHackMe

**Category:** Web / Directory Enumeration
**Platform:** TryHackMe — Hacker Holidays 2026

## Challenge Summary
The Byte Lotus guest-experience platform went live in a hurry, and the night-shift
developer shipped more than just the website — port 8080 was open with source code
exposure hiding behind it. The goal was to dump the exposed source and find the flag.

## Steps Taken

1. Installed OpenVPN for Linux and connected to the lab network, then opened the
   target site at `http://MACHINE_IP:8080`.

2. **What didn't work:** Manually inspected the website (viewing pages, checking
   visible source) but found nothing useful on the surface.

3. **Recon:** Ran `dirb` against the target to enumerate hidden paths and found a
   `.git/HEAD` file accessible on the server. Downloading it returned:
   ```
   ref: refs/heads/main
   ```
   This confirmed the `.git` directory itself was exposed on the live web server —
   meaning the entire repository history could potentially be reconstructed.

4. Ran `dirb` again against the base `.git/` path (without `HEAD`) and confirmed
   more git internals were exposed and downloadable.

5. **What worked:** Used `git-dumper` to reconstruct the full repository from the
   exposed `.git` folder:
   - Installed `git-dumper` from its GitHub repo inside a Python virtual environment
   - Located the `git-dumper` executable path with `which git-dumper`
   - Created a fresh directory for the recovered repo: `dumped-repo`
   - Ran:
     ```
     git-dumper http://10.114.159.8:8080/.git/ dumped-repo
     ```
   - This pulled down and reconstructed the exposed repository into `dumped-repo`

6. Inside the recovered repo, used `git log` to view commit history, then
   `git show <commit>` on the relevant commit to reveal the flag committed in
   plaintext.

## Root Cause

**Exposed `.git` directory on a production web server.** The developer deployed
the site without excluding the `.git` folder from the public web root. Since `.git`
contains the entire version history (not just the current live files), anyone who
finds it can reconstruct the full repo — including old commits that may contain
secrets, credentials, or flags that were later "removed" from the live code but
never scrubbed from history.

## Category Note
Classic **information disclosure via misconfiguration** — not an injection or logic
flaw, but a deployment mistake. A strong reminder that `git log`/`git show` on a
dumped repo can resurface anything ever committed, even if it's not in the current
HEAD of the visible site.
