# Towel on the Sunbed — TryHackMe

**Category:** Web Exploitation / Race Condition
**Platform:** TryHackMe — Hacker Holidays 2026

## Challenge Summary
Ponzi, a crypto rewards web app, gives users a daily reward with a 24-hour cooldown.
The goal was to find and exploit a flaw in how that cooldown is enforced, repeatedly
claim rewards, and reach "Whale" status to unlock the Whale Vault flag.

## Steps Taken

1. Logged into the Ponzi website with a newly created account.

2. **What didn't work:** Inspected the page and tried modifying the front-end
   JavaScript file (via browser DevTools) to bypass the cooldown check. This didn't
   work, because the cooldown validation was actually enforced **server-side** —
   front-end changes can't bypass a check the server itself performs.

3. **What worked:** Used Burp Suite to exploit a **race condition** in the
   reward-claiming logic:
   - Logged into Ponzi, opened Burp Suite, turned Intercept on
   - Clicked "Claim Reward" on the website, sending the request to Burp
   - Sent the intercepted request to **Repeater**, duplicated it into a group of 3
     identical requests
   - Used Repeater's **"Send group in parallel"** feature to fire all 3 requests to
     the server simultaneously
   - Turned Intercept off and refreshed the site — the **Whale Vault** option was
     now unlocked
   - Opened the Whale Vault and retrieved the flag

## Root Cause

A **race condition** vulnerability. When claiming a reward, the server likely:
1. Checks "has this user claimed in the last 24 hours?"
2. If not, grants the reward and updates the timestamp

Normally one request completes both steps before the next begins. But if multiple
requests hit the server at nearly the same instant, all of them can pass step 1
(the check) before *any* of them finishes step 2 (the update) — so all requests get
approved, even though only one should have been.

## Category Note
Distinct from standard OWASP Top 10 items (Injection, Broken Auth, etc.), though
broadly related to Broken Access Control — the app failed to properly restrict what
a user should be allowed to do.
