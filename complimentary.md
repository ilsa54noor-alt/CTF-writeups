# Complimentary — TryHackMe
**Category:** Cloud / AWS / Cognito / IAM Misconfiguration
**Platform:** TryHackMe — Hacker Holidays 2026

## Challenge Summary
The Byte Lotus Wellness app requires no login, yet somehow already knows guest
details the moment it's opened. Something behind the scenes is issuing AWS
credentials without proper checks. The goal was to find that mechanism, use the
credentials it hands out, and pull more than just my own record from the app's
backend data store.

## Steps Taken
1. Opened the target site (hosted on an S3 static website endpoint) and viewed the
   page source to look for clues about how the app worked under the hood.

2. Found a linked JavaScript file, opened it, and searched through it for anything
   related to AWS — found references worth noting for later (likely a Cognito
   Identity Pool ID or similar config baked into the front-end code).

3. Opened the browser dev console on the live page and typed `aws.config.credentials`
   to inspect what the app's AWS SDK had already loaded in-memory — this revealed
   temporary AWS credentials (access key, secret key, session token) that the app
   was using client-side.

4. Installed the AWS CLI locally, then ran:
   ```
   aws configure
   ```
   and entered the credentials pulled from the browser console.

5. **What worked:** With those credentials configured, queried the app's backend
   DynamoDB table directly:
   ```
   aws dynamodb scan --table-name complimentary-GuestWellnessProfiles
   ```
   This returned the **entire table** — not just my own guest record, but every
   guest's data, including the flag stored in another guest's profile.

## Root Cause

**Overly permissive IAM policy tied to unauthenticated Cognito Identity Pool access.**
The app used AWS Cognito to hand out temporary credentials to *any* visitor without
requiring login (that's how it "just knew things" — it was issuing real, functional
AWS credentials client-side to anonymous users). The IAM role attached to those
unauthenticated identities was scoped far too broadly — instead of restricting
access to only the requesting user's own record, it allowed a full `Scan` on the
entire DynamoDB table, exposing every guest's data to anyone who opened the app.

## Category Note
A textbook **cloud IAM misconfiguration** — the vulnerability wasn't in the app's
code logic but in how permissively AWS access was scoped. A strong reminder to
always check what temporary credentials handed to a client are actually capable of,
not just what the app's UI intends them to be used for.
