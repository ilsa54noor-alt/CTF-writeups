# Do Not Disturb — TryHackMe
**Category:** Boot2Root / Web
**Platform:** TryHackMe — Hacker Holidays 2026

## Challenge Summary
The Byte Lotus poolside platform tracks every cabana and sunbed session — but
someone else already has unauthorized access, having signed a wallet transaction
that wasn't theirs. Goal was to follow the intruder's footprints in, replicate the
access path, and recover both the user and root flags.

## Steps Taken
1. Connected to the lab machine over OpenVPN and ran an Nmap scan to enumerate open
   ports — found two open: SSH and HTTP. Focused on the web service first.
2. Opened the web app — a "Byte Lockers" / poolside cabana reservation page asking
   for a **staff or guest ID** and a **passphrase**, with no visible username or
   password given anywhere.
3. Checked the page source for hints — nothing obviously useful, aside from the
   input field placeholder text showing `attendant` as an example ID.
4. **What didn't work initially:** Tried basic SQL injection payloads in the login
   form using `attendant` as the username — got "invalid credentials" back. Also
   tried running `sqlmap` against the captured login request (saved via Burp Suite)
   at level 5 — ran into various tooling errors (malformed request file, 401
   unauthorized handling) which needed troubleshooting, but ultimately confirmed
   this wasn't a classic SQL injection.
5. **Recon:** Intercepted the login request in Burp Suite, inspected the JSON body,
   and recognized the structure resembled a **NoSQL (MongoDB) query** rather than
   SQL — the app was likely passing user input directly into a MongoDB query.
6. **What worked:** Used a NoSQL injection operator to bypass authentication.
   Modified the intercepted request so the password field became a MongoDB `$ne`
   (not-equal) operator instead of a literal string:
```json
   { "username": "attendant", "password": { "$ne": "aaaa" } }
```
   This makes the password check evaluate to "password is not equal to `aaaa`" —
   always true, regardless of the real password — bypassing the login entirely.
   Sending this returned a `302` redirect into the staff directory as `attendant`.
7. Logged into the "cabana desk" as `attendant` and found a **guest confirmation
   message customization** feature — a template field rendering guest name via
   what looked like an EJS templating engine.
8. Tested for **Server-Side Template Injection (SSTI)**: entered a math expression
   like `7*7` into the guest name field and previewed it. It returned `49` instead
   of the literal text — confirming the input was being evaluated as EJS template
   code rather than treated as plain text.
9. **What worked:** Searched for known EJS SSTI reverse shell payloads (using
   Node's `require('child_process')` / `global.process.mainModule.require` pattern),
   adapted one with the attacker IP and a listener port, started a `nc -lvnp <port>`
   listener, then submitted the payload through the template field and clicked
   Preview — this triggered the payload server-side and returned a shell.
10. Upgraded the shell to a proper TTY using a Python `pty.spawn()` one-liner
    (`python3 -c 'import pty; pty.spawn("/bin/bash")'`), since the raw shell had no
    job control.
11. Explored the filesystem, found the working directory (`/opt/poolside`), moved
    to `/home`, found users including `pipeline_svc` and `poolside`, and located the
    **user flag** inside the `poolside` user's home directory.

## Privilege Escalation (Two Stages)
**Stage 1 — poolside → pipeline_svc:**
1. Transferred `linpeas.sh` to the target by hosting a quick Python HTTP server
   locally (`python3 -m http.server 80`) and pulling it down on the target with
   `wget`.
2. Made it executable (`chmod +x linpeas.sh`) and ran it to enumerate privilege
   escalation vectors.
3. LinPEAS flagged a Node.js debug/inspect port (`9229`) listening only on
   localhost, tied to a `processor.js` script running as a different user
   (`pipeline_svc`) — a classic Node.js `--inspect` remote debugging exposure.
4. Connected to the exposed debugger from the target shell itself:
   debugfs /dev/<device>
3. Inside the `debugfs` shell, listed the contents of `/root`:
ls -l /root
4. Used `debugfs`'s `cat` equivalent to read `root.txt` directly off the raw disk
   — retrieving the **root flag** without ever needing actual root shell access,
   by reading the filesystem at the block-device level instead.

## Root Cause
Multiple chained vulnerabilities:
1. **NoSQL injection in authentication.** The login endpoint passed user-supplied
   JSON directly into a MongoDB query without sanitization, allowing operator
   injection (`$ne`) to bypass the password check entirely.
2. **Server-Side Template Injection (SSTI) in EJS.** User-controlled input (the
   guest confirmation message) was rendered directly through the EJS template
   engine instead of being treated as plain data, allowing arbitrary JavaScript/
   Node.js code execution on the server.
3. **Exposed Node.js debug port (`--inspect` on 9229).** A service was left running
   with remote debugging enabled and reachable, which effectively grants full code
   execution as that service's user to anyone who can reach the port — a serious
   and often-overlooked misconfiguration.
4. **World-readable raw block device.** The `pipeline_svc` user had permission to
   read the raw disk device directly, letting `debugfs` bypass normal file
   permission enforcement and read root-owned files straight off disk — a
   privilege escalation path that has nothing to do with typical sudo/SUID misconfigs.

## Category Note
A genuinely multi-stage **Boot2Root** chain: NoSQLi → SSTI → exposed Node debugger
→ raw-disk read as privesc. Each stage uses a completely different vulnerability
class, which makes this a strong example of how real intrusions often involve
lateral movement through several distinct weaknesses rather than one single exploit.
