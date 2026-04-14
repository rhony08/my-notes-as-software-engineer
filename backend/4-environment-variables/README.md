# Managing Configuration: Environment Variables and Secrets

One of the most common mistakes in backend development is hardcoding configuration values directly in the source code. This practice creates security vulnerabilities, makes deployments fragile, and prevents your application from running across different environments. In this article, we'll explore how to properly manage configuration using environment variables and secure secret management practices.

## Table of Contents

- [Why Hardcoded Configuration is Dangerous](#why-hardcoded-configuration-is-dangerous)
- [Environment Variables Basics](#environment-variables-basics)
- [The dotenv Pattern](#the-dotenv-pattern)
- [Environment-Specific Configuration](#environment-specific-configuration)
- [Secret Management](#secret-management)
- [The 12-Factor App Config Principle](#the-12-factor-app-config-principle)
- [Best Practices](#best-practices)
- [Conclusion](#conclusion)

## Why Hardcoded Configuration is Dangerous

Hardcoding configuration might seem convenient during development, but it creates serious problems:

### 1. Security Risks

**Database credentials in code:**
```javascript
// ❌ NEVER DO THIS
const dbConfig = {
  host: 'prod-db.example.com',
  user: 'admin',
  password: 'SuperSecret123!'
};
```

If this code is pushed to a public repository, your production database is compromised. Even in private repos, credentials in code violate the principle of least privilege.

### 2. Deployment Complexity

Hardcoded values mean you need different code versions for different environments:

```javascript
// ❌ Fragile approach
if (process.env.NODE_ENV === 'production') {
  apiUrl = 'https://api.production.com';
} else if (process.env.NODE_ENV === 'staging') {
  apiUrl = 'https://api.staging.com';
} else {
  apiUrl = 'http://localhost:3000';
}
```

### 3. Inflexibility

Changing a configuration value requires code changes, testing, and redeployment—even for simple URL or timeout adjustments.

## Environment Variables Basics

Environment variables are key-value pairs stored in the operating system environment, separate from your application code.

### Setting Environment Variables

**Linux/macOS:**
```bash
export DATABASE_URL="postgres://user:pass@localhost:5432/mydb"
export API_PORT=3000
export LOG_LEVEL=info
```

**Windows (Command Prompt):**
```cmd
set DATABASE_URL=postgres://user:pass@localhost:5432/mydb
set API_PORT=3000
```

**Windows (PowerShell):**
```powershell
$env:DATABASE_URL="postgres://user:pass@localhost:5432/mydb"
$env:API_PORT=3000
```

### Reading Environment Variables

**Node.js:**
```javascript
const dbUrl = process.env.DATABASE_URL;
const port = process.env.API_PORT || 3000; // Default value
```

**Python:**
```python
import os

db_url = os.environ.get('DATABASE_URL')
port = int(os.environ.get('API_PORT', 3000))
```

**Go:**
```go
import "os"

dbUrl := os.Getenv("DATABASE_URL")
port := os.Getenv("API_PORT")
if port == "" {
    port = "3000"
}
```

## The dotenv Pattern

The `dotenv` pattern loads environment variables from a `.env` file during development. This file is **never committed to version control**.

### How It Works

1. Create a `.env` file in your project root
2. Add your configuration values
3. Load it at application startup
4. Add `.env` to `.gitignore`

### Example .env File

```bash
# Database Configuration
DATABASE_URL=postgres://localhost:5432/myapp_dev
DATABASE_POOL_SIZE=10

# Application Settings
API_PORT=3000
NODE_ENV=development
LOG_LEVEL=debug

# External Services
REDIS_URL=redis://localhost:6379
SMTP_HOST=smtp.mailtrap.io
SMTP_PORT=2525

# Feature Flags
ENABLE_NEW_FEATURE=true
```

### Loading dotenv

**Node.js (dotenv package):**
```javascript
// At the very top of your entry file
require('dotenv').config();

// ES6 modules
import 'dotenv/config';
```

**Python (python-dotenv):**
```python
from dotenv import load_dotenv
load_dotenv()
```

**Go (godotenv):**
```go
import "github.com/joho/godotenv"

godotenv.Load()
```

### .env.example

Provide an `.env.example` file with dummy values so other developers know what to configure:

```bash
# Copy this file to .env and fill in real values
DATABASE_URL=postgres://user:password@localhost:5432/dbname
API_PORT=3000
REDIS_URL=redis://localhost:6379
JWT_SECRET=your-secret-key-here
```

## Environment-Specific Configuration

Different environments need different configurations:

### Configuration Structure

```
config/
├── default.json       # Base configuration
├── development.json   # Development overrides
├── staging.json       # Staging overrides
└── production.json    # Production overrides
```

### Environment-Based Loading

**Node.js (config package):**
```javascript
const config = require('config');

const dbConfig = config.get('database');
const apiPort = config.get('api.port');
```

**Using Environment Variables for Environment Detection:**
```javascript
const env = process.env.NODE_ENV || 'development';

const configs = {
  development: {
    database: { host: 'localhost', logging: true },
    api: { port: 3000 }
  },
  production: {
    database: { host: process.env.DB_HOST, logging: false },
    api: { port: process.env.PORT }
  }
};

module.exports = configs[env];
```

## Secret Management

Environment variables work for non-sensitive configuration, but secrets (passwords, API keys, tokens) need additional protection.

### Secret Management Solutions

**1. Cloud Provider Secret Managers:**
- AWS Secrets Manager
- Google Secret Manager
- Azure Key Vault

**2. Dedicated Tools:**
- HashiCorp Vault
- Doppler
- 1Password Secrets Automation

### Using AWS Secrets Manager

```javascript
const AWS = require('aws-sdk');
const secretsManager = new AWS.SecretsManager();

async function getDatabaseCredentials() {
  const secret = await secretsManager.getSecretValue({
    SecretId: 'prod/myapp/database'
  }).promise();
  
  return JSON.parse(secret.SecretString);
}
```

### Using HashiCorp Vault

```javascript
const vault = require('node-vault')({
  apiVersion: 'v1',
  endpoint: process.env.VAULT_ADDR
});

async function getSecret(path) {
  await vault.approleLogin({
    role_id: process.env.VAULT_ROLE_ID,
    secret_id: process.env.VAULT_SECRET_ID
  });
  
  const { data } = await vault.read(path);
  return data.data;
}
```

### Runtime Secret Injection

For containerized applications, secrets can be injected at runtime:

**Docker Compose:**
```yaml
services:
  app:
    environment:
      - DATABASE_URL
    secrets:
      - db_password

secrets:
  db_password:
    file: ./secrets/db_password.txt
```

**Kubernetes:**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
stringData:
  password: mysecretpassword
---
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
      - name: app
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: password
```

## The 12-Factor App Config Principle

The [12-Factor App methodology](https://12factor.net/config) provides excellent guidance:

> **Store config in the environment**
> 
> An app's config is everything that is likely to vary between deploys (staging, production, developer environments, etc.).

### Key Principles

1. **Strict separation of config from code**: Config varies across deploys, code does not
2. **No config groups**: Don't have "environments" as code concepts
3. **No config files in version control**: Use environment variables or external config stores
4. **All config via environment variables**: Easy to change between deploys without code changes

## Best Practices

### 1. Validate Configuration at Startup

Fail fast if required configuration is missing:

```javascript
const requiredEnvVars = ['DATABASE_URL', 'JWT_SECRET', 'API_PORT'];

for (const envVar of requiredEnvVars) {
  if (!process.env[envVar]) {
    console.error(`Missing required environment variable: ${envVar}`);
    process.exit(1);
  }
}
```

### 2. Use Type Conversion

Environment variables are always strings:

```javascript
// ❌ Wrong - port will be a string
const port = process.env.API_PORT || 3000;

// ✅ Correct - convert to number
const port = parseInt(process.env.API_PORT, 10) || 3000;

// ✅ Even better - use a validation library
const Joi = require('joi');

const envSchema = Joi.object({
  API_PORT: Joi.number().default(3000),
  LOG_LEVEL: Joi.string().valid('debug', 'info', 'warn', 'error').default('info'),
  DATABASE_URL: Joi.string().uri().required()
}).unknown().required();

const { error, value: envVars } = envSchema.validate(process.env);
if (error) throw new Error(`Config validation error: ${error.message}`);
```

### 3. Group Related Configuration

```javascript
const config = {
  database: {
    url: process.env.DATABASE_URL,
    poolSize: parseInt(process.env.DB_POOL_SIZE, 10) || 10,
    timeout: parseInt(process.env.DB_TIMEOUT, 10) || 5000
  },
  api: {
    port: parseInt(process.env.API_PORT, 10) || 3000,
    host: process.env.API_HOST || '0.0.0.0'
  },
  redis: {
    url: process.env.REDIS_URL
  }
};
```

### 4. Document Your Configuration

```markdown
## Environment Variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| DATABASE_URL | Yes | - | PostgreSQL connection string |
| API_PORT | No | 3000 | HTTP server port |
| LOG_LEVEL | No | info | Logging level (debug, info, warn, error) |
| REDIS_URL | No | - | Redis connection for caching |
```

### 5. Never Log Secrets

```javascript
// ❌ Dangerous - might log the full config object
console.log('Config:', config);

// ✅ Safe - explicitly select non-sensitive values
console.log('Config:', {
  port: config.api.port,
  environment: config.env,
  databaseHost: config.database.host // NOT the full URL with password
});
```

## Conclusion

Proper configuration management is fundamental to building secure, maintainable backend applications. By following these practices:

- **Use environment variables** for all configuration
- **Never commit secrets** to version control
- **Use .env files** for local development only
- **Validate configuration** at application startup
- **Use secret managers** for production secrets
- **Follow 12-Factor principles** for cloud-native apps

Your applications will be more secure, easier to deploy across environments, and simpler for other developers to work with.

Remember: Configuration is not code. Treat it as the external input that it is, and your applications will be more resilient and adaptable.