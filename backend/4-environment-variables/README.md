# Managing Configuration: Environment Variables and Secrets

Hardcoding database credentials in your code might seem convenient during development. Push that code to GitHub, and suddenly your production database is public. It's happened to companies of all sizes—including ones that should know better.

Environment variables are your first line of defense. They keep configuration separate from code, making your app more portable and your secrets... actually secret.

## Why This Matters

Every app has configuration that changes between environments:
- Database URLs (local vs staging vs production)
- API keys for third-party services
- Feature flags
- Debug modes
- Encryption keys

If these live in your codebase, you've got problems:
- **Security risk** — secrets get committed to version control
- **Deployment headaches** — you need different builds for different environments
- **Team friction** — everyone needs their own local config

## The 12-Factor Approach

The [12-Factor App methodology](https://12factor.net/config) puts it simply: **store config in environment variables**. This means:

- Config varies between deploys (dev, staging, prod)
- Code does not
- You can promote the same build through environments

## What Goes Wrong

### ❌ Hardcoded Secrets

```javascript
// NEVER DO THIS
const dbConfig = {
  host: 'prod-db.company.com',
  user: 'admin',
  password: 'supersecret123',
};
```

This gets committed to git. Anyone with repo access now has your production credentials.

### ❌ Config Files in Version Control

```javascript
// Still risky, even if it looks "organized"
const config = require('./config.json'); // config.json is in .gitignore... hopefully
```

Hope isn't a strategy. Someone *will* forget to add it to `.gitignore`, or remove it accidentally.

### ✅ Environment Variables

```javascript
// Clean, portable, secure
const dbConfig = {
  host: process.env.DB_HOST,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
};
```

No secrets in code. Same codebase works everywhere. Different environments just need different env vars.

## Getting Started

### Reading Environment Variables

Every language has built-in support:

**Node.js**
```javascript
const apiKey = process.env.API_KEY;
const port = parseInt(process.env.PORT || '3000', 10);
```

**Python**
```python
import os

api_key = os.environ.get('API_KEY')
port = int(os.environ.get('PORT', 3000))
```

**Go**
```go
package main

import (
    "os"
    "strconv"
)

func getEnv(key, fallback string) string {
    if value, exists := os.LookupEnv(key); exists {
        return value
    }
    return fallback
}

func main() {
    apiKey := getEnv("API_KEY", "")
    port, _ := strconv.Atoi(getEnv("PORT", "3000"))
}
```

### Setting Environment Variables

**Local Development**
```bash
# One-off
export DB_HOST=localhost
export DB_PASSWORD=localpass
node server.js

# Or inline
DB_HOST=localhost DB_PASSWORD=localpass node server.js
```

**Using .env Files**

Most frameworks support `.env` files for local development:

```bash
# .env
DB_HOST=localhost
DB_USER=dev
DB_PASSWORD=devpass
API_KEY=sk_test_123
```

```javascript
// Load .env file (using dotenv package)
require('dotenv').config();

// Now process.env has your .env values
console.log(process.env.DB_HOST); // localhost
```

**Important:** Add `.env` to `.gitignore` immediately:

```gitignore
# .gitignore
.env
.env.local
.env.*.local
```

### Production Environments

Cloud platforms handle this differently, but the principle is the same:

**Heroku**
```bash
heroku config:set DB_HOST=prod-db.company.com
heroku config:set DB_PASSWORD=supersecret
```

**AWS (ECS/Lambda)**
- Use AWS Systems Manager Parameter Store
- Or AWS Secrets Manager for rotation
- Environment variables reference the ARN

**Docker**
```bash
# Pass env vars at runtime
docker run -e DB_HOST=prod-db -e DB_PASSWORD=secret myapp

# Or use --env-file
docker run --env-file .env.prod myapp
```

**Kubernetes**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
    - name: app
      image: myapp:latest
      env:
        - name: DB_HOST
          value: "prod-db.company.com"
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
```

## Validation: Catch Missing Config Early

Missing environment variables cause confusing errors. Validate at startup:

```javascript
// config.js
const requiredEnvVars = [
  'DB_HOST',
  'DB_USER',
  'DB_PASSWORD',
  'API_KEY',
];

function validateEnv() {
  const missing = requiredEnvVars.filter(key => !process.env[key]);
  
  if (missing.length > 0) {
    console.error(`❌ Missing required environment variables: ${missing.join(', ')}`);
    process.exit(1);
  }
  
  console.log('✅ All required environment variables set');
}

module.exports = { validateEnv };
```

```javascript
// server.js
require('dotenv').config();
const { validateEnv } = require('./config');

validateEnv(); // Fail fast if config is missing

// Rest of your app...
```

## Secrets Management: Going Beyond Env Vars

Environment variables are great for config, but secrets (API keys, tokens, passwords) need extra care:

### The Problem with Env Vars

- Visible in process listings (`ps aux`)
- Visible in error logs if you're not careful
- Visible to anyone with shell access
- Hard to rotate without restarts

### Better Approaches

**AWS Secrets Manager / Parameter Store**
```javascript
const { SecretsManager } = require('@aws-sdk/client-secrets-manager');

async function getSecret(secretName) {
  const client = new SecretsManager();
  const response = await client.getSecretValue({ SecretId: secretName });
  return JSON.parse(response.SecretString);
}

// Use at startup, cache in memory
const dbPassword = await getSecret('prod/db-password');
```

**HashiCorp Vault**
```javascript
const vault = require('node-vault')({
  endpoint: process.env.VAULT_ADDR,
  token: process.env.VAULT_TOKEN,
});

async function getDbCredentials() {
  const { data } = await vault.read('database/creds/myapp');
  return {
    username: data.username,
    password: data.password,
  };
}
```

**Google Secret Manager / Azure Key Vault**
Similar patterns—fetch secrets at startup, cache in memory, rotate regularly.

## Trade-offs

| Approach | Pros | Cons | Use Case |
|----------|------|------|----------|
| Environment Variables | Simple, universal, 12-factor compliant | Visible in process list, logs | Non-sensitive config, development |
| .env Files | Easy local development | File management, still env vars | Local dev only |
| Secrets Manager | Encrypted, auditable, rotatable | Extra complexity, cost | Production secrets |
| Vault | Full-featured, self-hosted option | Operational overhead | Enterprise, multiple secret types |

## Common Mistakes

### ❌ Logging Environment Variables

```javascript
// NEVER do this
console.log('Config loaded:', process.env);
```

This dumps all secrets to logs. Anyone with log access sees everything.

### ✅ Safe Logging

```javascript
// Log what you need, not everything
console.log('Config loaded:', {
  nodeEnv: process.env.NODE_ENV,
  port: process.env.PORT,
  dbHost: process.env.DB_HOST, // Non-sensitive
  hasApiKey: !!process.env.API_KEY, // Boolean only
});
```

### ❌ Committing .env Files

Even with `.gitignore`, mistakes happen. Use a pre-commit hook:

```bash
# .githooks/pre-commit (or use husky)
if git diff --cached --name-only | grep -E '\.env$'; then
  echo "❌ Attempting to commit .env file!"
  exit 1
fi
```

### ❌ Using Env Vars for Everything

Not everything belongs in environment variables:

- **Large config objects** — use config files
- **Binary data** — use files or proper storage
- **Frequently changing values** — use a database or cache

## A Practical Setup

Here's a pattern that works well for most Node.js apps:

```
project/
├── .env.example      # Template with all required vars (committed)
├── .env              # Local secrets (gitignored)
├── config/
│   ├── index.js      # Config loading + validation
│   └── defaults.js   # Sensible defaults
└── .gitignore        # Includes .env
```

**.env.example** (committed to repo)
```bash
# Database
DB_HOST=localhost
DB_USER=myapp
DB_PASSWORD=your_password_here
DB_NAME=myapp_dev

# API Keys
API_KEY=your_api_key_here

# App Config
PORT=3000
NODE_ENV=development
```

**config/index.js**
```javascript
require('dotenv').config();

const required = ['DB_HOST', 'DB_USER', 'DB_PASSWORD', 'API_KEY'];
const missing = required.filter(key => !process.env[key]);

if (missing.length > 0) {
  console.error(`Missing required env vars: ${missing.join(', ')}`);
  console.error('Copy .env.example to .env and fill in values');
  process.exit(1);
}

module.exports = {
  database: {
    host: process.env.DB_HOST,
    user: process.env.DB_USER,
    password: process.env.DB_PASSWORD,
    name: process.env.DB_NAME || 'myapp',
  },
  api: {
    key: process.env.API_KEY,
  },
  app: {
    port: parseInt(process.env.PORT || '3000', 10),
    env: process.env.NODE_ENV || 'development',
  },
};
```

New developers clone the repo, copy `.env.example` to `.env`, fill in their values, and they're running.

## Key Takeaways

- **Never commit secrets** — use environment variables instead
- **Validate at startup** — fail fast if config is missing
- **Use .env files for local dev** — but keep them out of git
- **Log safely** — never dump `process.env` to logs
- **Consider secrets managers for production** — more secure than plain env vars
- **Provide .env.example** — helps new team members get started quickly

Configuration is one of those things that seems simple until it causes an incident. Get it right early, and you'll thank yourself later.