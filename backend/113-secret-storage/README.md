# Secret Storage: Vaults and Key Management

Somewhere in your org, there's a `.env.production` file with the database password, the Stripe key, and the S3 credentials sitting in a shared Slack channel — or worse, in a repo that was public for six months in 2021 and nobody ever checked the history. If that sounds familiar, this article is for you. Because the uncomfortable truth is: most "secret management" setups aren't managing secrets at all. They're just hiding them slightly better.

Let's talk about what actually goes wrong, and what a real secret storage setup looks like — from "good enough for a side project" to "auditors won't yell at you."

## First, What Counts as a Secret?

Anything that grants access if leaked:

- Database credentials (including connection strings with passwords baked in)
- API keys and tokens (Stripe, OpenAI, internal service auth)
- Private keys (TLS, signing keys for JWTs, SSH)
- Encryption keys themselves
- Third-party credentials (SES, Twilio, cloud provider keys)

The pattern to notice: they all have *blast radius*. A leaked dev database password is embarrassing. A leaked cloud root key is a crypto-mining bill and a compliance incident. So the first rule of secret storage is knowing where your secrets live in the first place — you can't protect what you can't inventory.

## Why `.env` Files Stop Being Enough

Let me be fair: for a side project, a `.env` file that's gitignored is fine. You're the only deployer, the blast radius is small, and the operational overhead of a vault is not worth it. The problems start when the team grows:

- **`.env` files multiply** — dev, staging, prod, each teammate's laptop. Which one is current? Nobody knows.
- **They get shared** — "hey can you send me the prod env?" is now a Slack message with production credentials in it.
- **They rot** — a credential revoked on one machine still works on the laptop of someone who left the company 8 months ago.
- **They're all-or-nothing** — the frontend dev who needs the API base URL gets the whole file, database password included.

```bash
# ❌ Secret in the repo, "hidden" by being in a config file
git add config/credentials.json
git commit -m "Add service credentials"

# ✅ Secret in the repo, encrypted - usable by CI but useless if leaked
sops --encrypt --kms arn:aws:kms:us-east-1:123456789012:key/abcd config/credentials.json
git add config/credentials.enc.json
git commit -m "Add encrypted service credentials"
```

That second option — **SOPS** or **git-crypt** — is the sweet spot for small teams. The secrets are encrypted at rest in the repo, decrypted at deploy time with a key only CI and your machines have. You get versioning, code review, and audit trail of *changes* (if not of *access*). It's a genuinely good middle ground between `.env` files and a full vault.

## The Real Deal: HashiCorp Vault

When you outgrow encrypted files, the industry default is Vault. It's a lot, but the core ideas are worth understanding even if you never run it:

### 1. Centralized storage + access policies

Secrets live in one place, and access is controlled by policy, not by "who has the file."

```
# policy: app-server.hcl
path "secret/data/backend/*" {
  capabilities = ["read", "list"]
}
path "secret/data/backend/db-root" {
  capabilities = ["deny"]   # app servers never see the root credential
}
```

An app gets a token (or a short-lived JWT from Kubernetes auth) that grants *only* what it needs. The database password is visible to the backend service, not to the data team, not to the frontend devs, not to the person who just joined.

### 2. Dynamic secrets — the killer feature

Static secrets (even in a vault) still get shared, still get leaked, still live for years. Vault's real superpower is **dynamic secrets**: it talks to the database or cloud provider and *creates* a credential on demand, with a lease.

```bash
# Vault creates a DB user with a TTL of 1 hour, scoped to read-only
$ vault read database/creds/readonly
Key                Value
---                -----
lease_id           database/creds/readonly/9p3s...
lease_duration     1h
password           s3cr3t-that-expires
username           v-token-readonly-9p3s
```

The app uses that credential for an hour, then it's gone. Revoked automatically. If it leaks, the blast radius is *one hour of read-only access*. Compare that to a production password that's been valid since 2019. This is the single biggest upgrade you can make to your secret posture, and it works with databases, AWS/GCP, and SSH.

**Trade-off alert:** dynamic secrets mean your app needs to *fetch* credentials at startup (and handle lease renewal), which is more code and more failure modes. If your DB user gets created at 9:00 and the app crashes at 9:30, on restart it needs a *new* credential — startup gets slower and needs retry logic. For many teams, static secrets in Vault with aggressive rotation is the pragmatic choice.

### 3. Encryption as a service (Transit)

Vault can also *hold* encryption keys and encrypt/decrypt data without the key ever leaving the vault. Apps send plaintext, get ciphertext back. This kills the "where do we store the encryption key for the encryption keys?" problem — the key never exists in your app's memory.

## KMS and Envelope Encryption

Cloud providers solve a slightly different problem: **key management**. AWS KMS (and GCP KMS, Azure Key Vault) give you hardware-backed keys that only the cloud service can use. You can't export them — you can only ask KMS to encrypt or decrypt.

The pattern you'll see everywhere is **envelope encryption**:

1. Your app asks KMS for a *data key* (a plaintext key + an encrypted copy)
2. The plaintext data key encrypts your actual data (fast, local, symmetric)
3. The encrypted data key is stored alongside the data
4. To decrypt: KMS decrypts the data key, then the data key decrypts the data

```js
// ❌ Single master key for everything - one compromise, total compromise
const masterKey = getMasterKeyFromEnv();
const ciphertext = aes256Encrypt(masterKey, userData);

// ✅ Envelope encryption via KMS
const { Plaintext, CiphertextBlob } = await kms.generateDataKey({
  KeyId: process.env.KMS_KEY_ID,
  KeySpec: 'AES_256'
});
const ciphertext = aes256Encrypt(Plaintext, userData);   // fast, local
await store(userId, ciphertext, CiphertextBlob);          // store encrypted key too

// later: KMS.decrypt(CiphertextBlob) -> Plaintext -> decrypt userData
```

Why bother? Because KMS calls are rate-limited and cost money — you don't want one per record. Envelope encryption means you pay for one KMS call per *session* (to unwrap the data key) and encrypt everything else with fast local AES. Also, you can rotate the master key in KMS without re-encrypting all your data — old data keys still decrypt with the old master version.

## Managed Secret Managers: The Pragmatic Default

Honestly? For most teams, **AWS Secrets Manager** or **GCP Secret Manager** is the right answer, not a self-hosted Vault. You get:

- Storage + versioning + audit logging (who read what, when — baked in)
- Automatic rotation for some credential types (RDS passwords rotate on a schedule)
- IAM-based access control — your existing identity system, not a new one
- Zero ops — no Vault cluster to patch, upgrade, and keep highly available

```js
// App fetches the DB password at startup - never in env, never in code
const secret = await secretsManager.getSecretValue({ SecretId: 'prod/db/main' });
const { username, password } = JSON.parse(secret.SecretString);
pool = createPool({ host: dbHost, user: username, password, ... });
```

The trade-off is real though: you're locked into that cloud's ecosystem, and you don't get dynamic secrets (Secrets Manager stores static values — it rotates them, but the credential still *exists* and is shared). Vault's dynamic creds are a fundamentally stronger model. The question is whether your threat model justifies running a Vault cluster.

**Decision shortcut:**

| Approach | Best for | Cost | Gotcha |
|----------|----------|------|--------|
| `.env` + gitignore | Solo projects, prototypes | Free | No sharing, no audit, easy to mess up |
| SOPS / git-crypt | Small teams, few services | Low | Secrets still static; decryption key is a SPOF |
| Cloud Secret Manager | Most production teams | Low-med | Cloud lock-in, no dynamic creds |
| HashiCorp Vault | Regulated, multi-team, dynamic creds | High (ops) | You operate it; HA + unsealing are real work |
| KMS + envelope | Data encryption at rest | Med | Doesn't store app secrets — solves a different problem |

## Rotation: The Thing Nobody Does

Here's a test: when was the last time you rotated the production database password? If the answer is "never" or "I don't know," you have a static secret with an unknown lifetime — the worst kind. Rotation is where good secret storage pays for itself, because the tooling *makes* it easy:

- **Vault dynamic secrets:** rotation is automatic — new creds on every lease
- **Secrets Manager:** scheduled rotation for supported types
- **Static secrets:** you need a process — rotate on a schedule, on suspected leak, and when anyone with access leaves

```bash
# Rotation checklist for a manual rotation:
# 1. Create new credential in the source system (DB, cloud, provider)
# 2. Store it in the secret manager (new version)
# 3. Deploy apps so they pick up the new version
# 4. Verify apps work
# 5. Revoke the OLD credential
# 6. Confirm old credential is dead (try using it - it should fail)
```

Step 6 is the one everyone skips. If you never verify the old credential is dead, you don't actually know it's rotated.

## What Breaks If You Ignore All This

- A leaked key in a public repo gets scanned by bots within minutes — GitHub's secret scanning will alert you *after* it's already been indexed and harvested
- No audit trail means a disgruntled ex-employee with a copy of `.env` is undetectable — you'll find out when the invoice arrives
- Static credentials shared across environments mean "rotate prod" also breaks staging, and nobody knows which services use which secret
- An encryption key stored next to the data it encrypts is not encryption — it's obfuscation with extra steps

## Takeaways

- **Inventory your secrets first** — you can't protect what you can't find. A quick `grep -r "password" .` through your repos is a great Friday afternoon activity.
- **`.env` files are fine for toys, not teams** — the moment a second person needs credentials, move to SOPS or a secret manager.
- **Prefer managed secret managers over self-hosted Vault** unless you specifically need dynamic credentials or have compliance requirements — the ops cost of Vault is real.
- **If you do run Vault, use dynamic secrets** — that's the feature that makes it worth it; static secrets in Vault are just `.env` files with extra steps.
- **Use envelope encryption with KMS for data at rest** — one KMS call per session, not per record.
- **Rotate on a schedule, and verify old credentials are actually dead** — rotation you can't prove is rotation you didn't do.
- **Never put a secret in a commit message, log line, or error message** — `console.log(secret)` in an error handler is how leaks happen at 2am.
