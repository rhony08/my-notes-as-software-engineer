# Kubernetes ConfigMaps and Secrets

Every app has configuration. Database URLs, API keys, feature flags, environment-specific settings. You *could* hardcode them in your container image. But then you're rebuilding and redeploying for every env, and your secrets are sitting in a registry somewhere.

That's where ConfigMaps and Secrets come in. They're Kubernetes' way of decoupling configuration from your application code.

## ConfigMaps: The Basics

A ConfigMap is exactly what it sounds like—a map of configuration data. Key-value pairs, plain text, meant for non-sensitive stuff.

Creating one is straightforward:

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_ENV: production
  LOG_LEVEL: info
  API_URL: https://api.example.com
  MAX_CONNECTIONS: "100"  # Note: it's a string, not an int
```

Or imperatively:

```bash
kubectl create configmap app-config --from-literal=APP_ENV=production --from-literal=LOG_LEVEL=info
```

You can also create them from files:

```bash
# Creates a ConfigMap with file contents as values
kubectl create configmap app-config --from-file=config.properties --from-file=app.config
```

## Secrets: Same Idea, Different Handling

Secrets look almost identical to ConfigMaps, but they're intended for sensitive data:

```yaml
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque  # default type for arbitrary key-value pairs
data:
  DB_PASSWORD: c3VwZXJzZWNyZXQ=  # base64 encoded
  API_KEY: c2stdGVzdDEyMzQ1Njc4OTA=
```

The values **must be base64 encoded** in YAML. Or use `stringData` for convenience (Kubernetes encrypts it under the hood):

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
stringData:
  DB_PASSWORD: supersecret
  API_KEY: sk-test1234567890
```

### 🚨 Important: Base64 is NOT encryption

This trips up a lot of people. Let's be very clear:

❌ **"It's base64, so it's secure, right?"**

No. Base64 is encoding, not encryption. Anyone with `kubectl get secret` access can decode it instantly:

```bash
kubectl get secret app-secrets -o jsonpath='{.data.DB_PASSWORD}' | base64 --decode
# -> supersecret  😬
```

✅ **What you should actually do:**
- Enable **encryption at rest** on your cluster (etcd encryption)
- Use an external secrets solution like **Vault**, **AWS Secrets Manager**, or **Sealed Secrets**
- Never commit raw secrets YAML to git

## How to Use Them: Three Methods

### 1. Environment Variables

The most common approach. Your app reads `process.env` or `os.Getenv()` and off you go.

```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
      - name: app
        image: myapp:latest
        env:
          # From a ConfigMap key
          - name: APP_ENV
            valueFrom:
              configMapKeyRef:
                name: app-config
                key: APP_ENV
          # From a Secret key
          - name: DB_PASSWORD
            valueFrom:
              secretKeyRef:
                name: app-secrets
                key: DB_PASSWORD
```

Or inject **all keys at once** (useful for settings files):

```yaml
envFrom:
  - configMapRef:
      name: app-config
  - secretRef:
      name: app-secrets
```

**Trade-off:** `envFrom` is convenient but makes it hard to trace where values come from. I've debugged production issues where a ConfigMap key silently overwrote an env var the app expected. Be explicit when the key naming matters.

### 2. Volume Mounts

Sometimes env vars aren't the right fit. Maybe your app reads a config file, or you need the values updated without restarting the pod.

```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
      - name: app
        image: myapp:latest
        volumeMounts:
          - name: config-volume
            mountPath: /etc/config
          - name: secret-volume
            mountPath: /etc/secrets
            readOnly: true
      volumes:
        - name: config-volume
          configMap:
            name: app-config
        - name: secret-volume
          secret:
            secretName: app-secrets
```

With volumes, each key becomes a file:

```
/etc/config/
├── APP_ENV
├── LOG_LEVEL
├── API_URL
└── MAX_CONNECTIONS

/etc/secrets/
├── DB_PASSWORD
└── API_KEY
```

**Why choose volumes over env vars?**

| Method | Updates without restart | Max value size | Use case |
|--------|------------------------|----------------|----------|
| Env vars | No (pod restart needed) | ~1MB per ConfigMap | Small, static config |
| Volume mounts | Yes (eventual, ~sync interval) | ~1MB per ConfigMap | Config files, dynamic settings |

✅ **Pro tip for volume mounts:** ConfigMaps mounted as volumes are updated periodically (controlled by `--sync-interval` on kubelet, default 60s). But env vars are set at pod start and never change. If you want live config updates without restarting, use volume mounts.

### 3. Command-line Arguments

Less common, but useful for tools or init containers:

```yaml
containers:
- name: app
  image: myapp:latest
  command: ["sh", "-c"]
  args:
    - |
      /app/start --db-url=$(DB_URL) --log-level=$(LOG_LEVEL)
  env:
    - name: DB_URL
      valueFrom:
        secretKeyRef:
          name: app-secrets
          key: DB_URL
    - name: LOG_LEVEL
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: LOG_LEVEL
```

## Real-World Patterns

### Pattern 1: Layered Configuration

One pattern I've seen work well: use ConfigMaps for env-specific defaults, Secrets for sensitive values, and let your app override with env vars at the container level.

```
ConfigMap (defaults for staging/prod)
  └── Override with container env
       └── Secret values injected separately
```

### Pattern 2: Sealed Secrets

If you're committing Secrets to git (which you shouldn't do raw), use tools like **Bitnami Sealed Secrets**:

```yaml
# This is safe to commit!
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: app-secrets
spec:
  encryptedData:
    DB_PASSWORD: AgBYJ7... (encrypted with cluster key)
```

The SealedSecret controller decrypts it into a regular Secret in your cluster. Only your cluster can decrypt it, so it's safe in version control.

### Pattern 3: External Secrets Operator

For cloud-native setups, the **External Secrets Operator** pulls directly from cloud providers:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: app-secrets
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secretsmanager
    kind: SecretStore
  target:
    name: app-secrets
  data:
    - secretKey: DB_PASSWORD
      remoteRef:
        key: /prod/app/db-password
```

No raw Secrets YAML anywhere. Just references to the external store.

## Common Gotchas

### Data mutations are not tracked

Change a ConfigMap? Pods with volume mounts will pick it up eventually. Pods with env vars? Nope. There's no built-in rollout trigger.

✅ **Solution:** Use tools like `reloader` (stakater/Reloader) that watch ConfigMap/Secret changes and trigger rolling updates.

### ConfigMap size limits

ConfigMaps are stored in etcd, which has a default max request size of 1.5MB. Your ConfigMap + all its keys plus metadata needs to fit. Don't try to store large files here.

### String vs non-string values in env vars

Everything in a ConfigMap is a string. So `MAX_CONNECTIONS: 100` becomes the string `"100"`, not the integer `100`. Most apps handle this fine, but it's caught me with strict type-checking languages.

### Secrets are stored in etcd unencrypted by default

Unless you've enabled encryption at rest, Secrets are stored as plain text in etcd. Enable it:

```yaml
# kube-apiserver encryption config
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: <base64-encoded-32-byte-key>
      - identity: {}
```

## Takeaways

- **ConfigMaps** are for non-sensitive config; **Secrets** are for sensitive data — but both are stored in etcd
- Base64 is not encryption — treat your Secrets YAML like you'd treat plaintext passwords
- Use **env vars** for simple, static config; use **volume mounts** for live updates or config files
- Never commit raw Secrets to git — use Sealed Secrets or External Secrets Operator
- Watch out for etcd size limits and the stringification of all ConfigMap values
- Enable encryption at rest in your cluster, or your "secrets" are just mildly obfuscated plaintext
