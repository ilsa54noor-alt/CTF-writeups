# Packed Light — TryHackMe
**Category:** Forensics / Network Forensics / PCAP Analysis / Cryptography
**Platform:** TryHackMe — Hacker Holidays 2026

## Challenge Summary
A short packet capture from the guest network showed small, suspiciously regular
outbound requests data quietly being exfiltrated on a loop, hidden inside traffic
that looked ordinary at a glance. The goal was to identify the covert channel,
reassemble the exfiltrated data, and decode it to get the flag.

## Steps Taken
1. Downloaded and extracted the provided task zip file, which contained a Wireshark
   capture (`.pcapng`).
2. Opened the capture in Wireshark and filtered on `http` traffic to narrow down
   what was actually being sent.
3. Found a Python script referenced in the traffic (an "updates" file) and examined
   it — it revealed how the app was building its outbound requests:
```python
   b64_string = base64.b64encode(encrypted).decode('utf-8')

   headers = {
       "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) ByteLotusClient/1.1",
       "Cookie": f"hotel_sess_state={b64_string}"
   }
```
   This showed the exfiltration method: encrypted data → base64 encoded → smuggled
   out inside an innocent-looking `Cookie` header (`hotel_sess_state`), disguised as
   normal browser/session traffic.
4. In Wireshark, added the `Cookie` field as a custom column to make the relevant
   values easy to spot across all the captured requests.
5. Used `tshark` to extract the cookie values directly from the command line:
   tshark -r traffic.pcapng -Y 'http.cookie contains "hotel_sess_state"' -T fields -e http.cookie
   This pulled out the `hotel_sess_state` value being sent in each request.
6. **What worked:** Took the extracted base64 cookie value into CyberChef to decode
   and decrypt it. The script also revealed how the encryption key was constructed:
```python
   def getkey():
       p1 = "H0t3lSt@ff0Nly"
       p2 = "K3epS3cr3t!"
       return p1 + p2
```
   Using this reconstructed key in CyberChef's recipe (base64 decode, then decrypt
   with the derived key) revealed the flag hidden inside the "session cookie."

## Root Cause
**Data exfiltration disguised as legitimate HTTP traffic.** The malicious script
encrypted data locally, base64-encoded it, then smuggled it out inside an HTTP
`Cookie` header — a field most casual traffic inspection ignores because it looks
like normal session-handling behavior. The encryption key itself was hardcoded and
split into two string fragments inside the script, which is weak obfuscation at
best — trivial to reconstruct once the source was found.

## Category Note
A good example of **covert channel exfiltration** — the "vulnerability" here isn't
in a web app but in traffic analysis: recognizing that a field which looks routine
(a cookie) can carry an entire encrypted payload. Combines PCAP analysis, script
reverse engineering, and cryptography in one challenge.
