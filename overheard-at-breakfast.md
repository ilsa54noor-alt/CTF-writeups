# Overheard at Breakfast — TryHackMe

**Category:** OSINT / Social Media / Hashing
**Platform:** TryHackMe — Hacker Holidays 2026

## Challenge Summary
A leaked screenshot of a chat conversation between two hotel guests ("Ponzi" and
"Lambo") hinted at a now-abandoned social media/profile-linking account. Goal was
to read the conversation carefully for identifying details, track down the hidden
account, and recover the flag.

## Steps Taken

1. Opened the provided task file and read through the full chat transcript between
   Ponzi and Lambo rather than just skimming it — the room's OSINT hint from
   `@0xMia` specifically warned that the key detail was easy to miss if you didn't
   actually read the conversation closely.
2. Picked out the relevant details from Lambo's messages:
   - Mentioned using a **free tool** to upload a profile and link other social
     media accounts, which they said **"started with a G"** — but couldn't recall
     the exact name.
   - Shared their email directly in the chat: `LambobyteLotushotel@gmail.com`
3. Took the "started with a G" clue and asked an AI assistant what free profile-
   linking tool that could be — it identified **Gravatar** (a service that generates
   a public profile page tied to a hashed version of a user's email address).
4. **What worked:** Went to Gravatar and looked up the profile tied to
   `LambobyteLotushotel@gmail.com`. This returned a **hashed value** (Gravatar
   generates profile URLs using an MD5/SHA256 hash of the associated email).
5. Took that hashed value into **CyberChef**, ran it through a **Base64 decode**
   recipe, and recovered the flag hidden inside.
   
## Root Cause
This wasn't a technical vulnerability in an app — it's a classic **OSINT /
information leakage** scenario. Lambo unknowingly leaked two connectable pieces of
information in casual conversation: a direct email address, and a hint toward a
profile-linking service tied to that email. Gravatar profiles are public by design
(the whole point of the service is to be an internet-wide avatar/identity linked to
an email's hash), so once the email was known, the "hidden" account wasn't actually
hidden at all — it was one hash lookup away.

## Category Note
A good reminder that **OSINT investigations often hinge on connecting small,
throwaway details** rather than any single "aha" clue — an email mentioned in
passing, plus a half-remembered service name, was enough to fully deanonymize an
account. Also a solid intro to how Gravatar's hash-based profile system works and
why sharing your email casually can expose more than you'd expect.
