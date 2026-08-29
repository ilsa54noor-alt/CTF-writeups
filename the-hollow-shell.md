# The Hollow Shell — TryHackMe

**Category:** Web / Zip Slip / Path Traversal
**Platform:** TryHackMe — Hacker Holidays 2026

## Challenge Summary
The Byte Lotus "Shoreline Display" portal lets staff upload decorative "shells"
(zip packages containing display assets) that get automatically extracted and
displayed in guest rooms. The goal was to find a way past what the upload/extraction
process forgets to check, and turn a shell upload into a real shell on the box.

## Steps Taken
1. Browsed to the target IP directly — got an error, so ran an Nmap scan to
   enumerate open ports. Found two open: **SSH** (no credentials available, so set
   aside) and **port 5000**.
2. Navigated to `http://MACHINE_IP:5000` and landed on a **"Shoreline Display"
   staff signing portal** — a login page asking for a staff ID and passphrase.
3. **Recon:** Viewed the page source and found **developer credentials left behind
   in plain text** — a classic leftover-in-source-code disclosure.
4. Logged in with the leaked credentials and reached the **Room Service /
   dashboard**, which had an upload feature: **"Bring a shell ashore"** — accepting
   only `.zip` files. The upload description noted that shells "may include
   optional automation hooks" that the "theme system applies shortly after the
   shell comes ashore" — a strong hint that some extracted content gets
   automatically executed or processed after upload.
5. Learned the required structure: every uploaded zip needed a `shell.json`
   manifest file inside it listing its assets. Built a minimal valid one:
```json
   { "name": "beach" }
```
   Zipped it and uploaded it successfully to confirm the basic flow worked — the
   file was extracted and made accessible at a shell-specific URL path
   (`/shells/<id>/...`).
6. **Vulnerability identified — Zip Slip.** Recognized this as a classic **Zip
   Slip** vulnerability: a path traversal flaw in the zip extraction logic, where
   filenames inside the archive containing `../` sequences aren't sanitized before
   extraction. This lets an attacker write files **outside** the intended
   extraction directory — including into a special `hooks/` directory two levels
   up from where shells normally get extracted, which the "automation hooks" hint
   implied gets executed automatically.
7. **What worked:** Built a malicious zip using a Python script that crafted a zip
   archive containing:
   - A valid `shell.json` (to pass the manifest check)
   - A normal asset file (`image.png`) to look legitimate
   - A path-traversal-crafted entry (`../../hooks/shell.py`) containing a Python
     reverse shell payload, designed to land in the automation hooks directory
     during extraction instead of the shell's own sandboxed folder
   Updated the script with the correct attacker IP (from `ifconfig`/`tun0`) in both
   required spots, then ran it to generate `payload.zip`.
8. Started a netcat listener (`nc -lvnp <port>`) locally, then uploaded
   `payload.zip` through the portal's upload form.
9. Once uploaded, the server's automation hook mechanism picked up and executed the
   planted `hooks/shell.py`, triggering the reverse shell payload back to the
   listener — giving full shell access on the target.
10. Explored the filesystem: listed the home directory, found two users (`ubuntu`
    and `roomservice`), moved into `roomservice`'s home directory, and located the
    **flag**.

## Root Cause
**Zip Slip (archive extraction path traversal).** The shell upload/extraction logic
trusted filenames stored inside the uploaded zip without validating or sanitizing
them for directory traversal sequences (`../`). Combined with an "automation hooks"
feature that automatically executed files found in a specific `hooks/` directory,
this let an attacker escape the intended per-shell sandbox directory during
extraction and drop an executable payload directly into a location the server
would run automatically — turning a simple file upload feature into full remote
code execution.

## Category Note
A textbook example of **CWE-22 (Path Traversal)** applied specifically to archive
extraction — often called **Zip Slip**, a vulnerability class that's shown up in
many real-world libraries (Java, Node.js, Python, Go) over the years. The key
lesson: any feature that extracts an archive server-side must validate that every
extracted file path stays within the intended output directory, and any
"auto-run on upload" feature is an especially dangerous combination with unsafe
extraction.
