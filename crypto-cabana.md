# CryptoCabana — TryHackMe

**Category:** Cloud / Azure / Storage / Key Vault
**Platform:** TryHackMe — Hacker Holidays 2026

## Challenge Summary
The CryptoCabana kiosk promised "Backed up. Sleep easy." for guest seed phrase
backups. Goal was to find out what the kiosk's front-end quietly trusts to reach
into Azure storage on its own, follow that trust chain, and recover secrets from a
Key Vault that doesn't give up real values on the first attempt.

## Steps Taken
1. Set up the Azure CLI environment via the provided Azure Cloud Shell (logged in
   with the given temporary credentials, opened Bash, selected the CTF subscription,
   validated access with `az account show`).
2. Opened the target kiosk site and entered random/dummy credentials into the
   "back it up" form to observe what happened, while watching the browser's
   Network tab — saw a `PUT` request going out to an `app.js`-related endpoint.
3. **Recon:** Viewed the page source / used the browser debugger to pull up `app.js`
   directly. Found hardcoded credentials in the front-end JavaScript: an **Azure
   storage account name**, a **backup container name**, and a **SAS token**
   (Shared Access Signature) — meant to let the kiosk write backups to storage
   without a real login, but exposed to anyone who inspected the client-side code.
4. Used the Azure CLI with the leaked SAS token to list available containers:
   az storage container list --account-name <ACCOUNT> --sas-token "<TOKEN>" --output table
   Found three containers: `web`, `backups`, and `vault`.
5. Listed blobs inside each container:
az storage blob list --account-name <ACCOUNT> --container-name <CONTAINER> --sas-token "<TOKEN>" --output table
   - `web` and `backups` returned nothing useful (or argument errors needing minor
     command syntax fixes).
   - `vault` contained two files: `backup_service_account.json` and `seedphrase.txt`.
6. Downloaded both files from the `vault` container:

az storage blob download --account-name <ACCOUNT> --container-name vault --name backup_service_account.json --sas-token "<TOKEN>" --file wa_file
az storage blob download --account-name <ACCOUNT> --container-name vault --name seedphrase.txt --sas-token "<TOKEN>" --file wa_file2

7. Read the downloaded files:
   - `backup_service_account.json` contained a **service principal** — client ID,
     client secret, tenant ID, and the target **Key Vault name/URL** — labeled
     "rotate this if it ever leaves the vault."
   - `seedphrase.txt` contained a decoy/red herring seed phrase (a set of
     unrelated-looking words), not the actual flag path.
8. **What worked:** Authenticated to Azure as the leaked service principal:
az login --service-principal --username <CLIENT_ID> --password <CLIENT_SECRET> --tenant <TENANT_ID>
   Confirmed the account type switched from `user` to `servicePrincipal`.
9. Listed secrets inside the Key Vault:
az keyvault secret list --vault-name <VAULT_NAME> --output table
   Found three secrets: `key-shard-1`, `key-shard-2`, `key-shard-3`, plus a
   `master-key`.
10. Read each shard to reconstruct the flag:
az keyvault secret show --vault-name <VAULT_NAME> --name key-shard-1
    - `key-shard-1` returned cleanly — first part of the flag.
    - `key-shard-2` returned an error: the secret had been **rotated** (current
      value no longer valid), with a note that the old value should still be
      recoverable if you know where to look.
    - Listed all historical versions of that secret:
  az keyvault secret list-versions --vault-name <VAULT_NAME> --name key-shard-2
      Grabbed the previous version's ID and read that specific version directly:
  az keyvault secret show --vault-name <VAULT_NAME> --name key-shard-2 --version <VERSION_ID>
      This returned the second part of the flag.
    - `key-shard-3` was read the same way as shard 1 (no rotation needed) to get
      the final part.
11. Combined all three shards to assemble the complete flag.

## Root Cause
A **cloud trust chain misconfiguration**, layered across multiple services:
1. **Client-side secrets exposure.** The kiosk's front-end JavaScript hardcoded a
   live Azure Storage SAS token, meant only for the app's own backend use, but
   fully readable by any visitor via browser dev tools.
2. **Overly broad SAS token permissions.** That token wasn't scoped to just the
   intended `backups` container — it also granted access to a `vault` container
   never linked anywhere in the kiosk's own UI, exposing service principal
   credentials that should have stayed server-side only.
3. **Secret versioning left old values recoverable.** Rotating a Key Vault secret
   doesn't delete prior versions by default — so even a "safely rotated" credential
   remained fully retrievable by anyone who could enumerate secret versions,
   defeating the purpose of the rotation.

## Category Note
A strong example of **cloud privilege chaining**: a leaked low-privilege token
(SAS) led to a service principal, which led to Key Vault access, which — even after
"rotation" — still leaked historical secret versions. Each step alone looks minor;
chained together, they add up to full compromise. Also a good reminder that secret
rotation without version cleanup provides a false sense of security.
