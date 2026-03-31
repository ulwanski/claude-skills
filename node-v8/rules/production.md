---
name: production
description: Production vs development configuration, NODE_ENV, 12-factor patterns
metadata:
  tags: production, node-env, configuration, deployment, best-practices
---

# Node.js Production Configuration

## NODE_ENV — always set to "production"

Node.js itself has no special production mode, but many npm packages (Express, etc.)
check `NODE_ENV` to enable or disable optimizations (caching, detailed error messages,
minification, etc.).

```bash
# Always run with this set in production
NODE_ENV=production node app.js

# In Docker
ENV NODE_ENV=production

# In systemd
Environment=NODE_ENV=production
```

## NODE_ENV as antipattern — avoid branching on it in your own code

The official Node.js docs consider mixing environment name with behavior an antipattern.
It makes staging and production behave differently, breaking reliable testing.

```javascript
// BAD: behavior differs between environments
if (process.env.NODE_ENV === 'development') {
  app.use(morgan('dev'));
}
if (process.env.NODE_ENV === 'production') {
  app.use(compression());
}
// staging gets neither!

// GOOD: use explicit feature flags instead
const LOG_REQUESTS  = process.env.LOG_REQUESTS  === 'true';
const USE_COMPRESSION = process.env.USE_COMPRESSION !== 'false'; // default on

if (LOG_REQUESTS)     app.use(morgan('combined'));
if (USE_COMPRESSION)  app.use(compression());
```

## 12-factor app configuration

Store all environment-specific config in environment variables (never in code):

```javascript
// BAD: hardcoded config
const db = new Database('postgres://localhost:5432/mydb');
const PORT = 3000;

// GOOD: environment variables with sensible defaults
const db = new Database(process.env.DATABASE_URL);
const PORT = parseInt(process.env.PORT ?? '3000', 10);
```

Validate at startup — fail fast rather than silently:

```javascript
const required = ['DATABASE_URL', 'SECRET_KEY', 'REDIS_URL'];
for (const key of required) {
  if (!process.env[key]) {
    console.error(`Missing required env var: ${key}`);
    process.exit(1); // fail immediately, not during first request
  }
}
```

## Production-specific Node.js flags

```bash
# Limit memory (avoid OOM silently eating all RAM)
node --max-old-space-size=512 app.js

# Expose GC metrics
node --expose-gc app.js

# Generate heap snapshot on OOM instead of crashing silently
node --heapsnapshot-near-heap-limit=3 app.js
```

## Process management

Don't run Node.js as PID 1 in Docker — it doesn't handle signals correctly:

```dockerfile
# BAD
CMD ["node", "app.js"]

# GOOD: use a proper init system
CMD ["dumb-init", "node", "app.js"]
# or use tini:
# ENTRYPOINT ["/sbin/tini", "--"]
```

Handle graceful shutdown:

```javascript
function shutdown(signal) {
  console.log(`Received ${signal}, shutting down gracefully`);
  server.close(() => {
    db.end();       // close DB pool
    process.exit(0);
  });
  // Force shutdown after 30s
  setTimeout(() => process.exit(1), 30_000).unref();
}

process.on('SIGTERM', () => shutdown('SIGTERM'));
process.on('SIGINT',  () => shutdown('SIGINT'));
```

## Error handling in production

```javascript
// Catch unhandled promise rejections — they will terminate the process in Node.js 15+
process.on('unhandledRejection', (reason, promise) => {
  console.error('Unhandled Rejection at:', promise, 'reason:', reason);
  // Log to your error tracker (Sentry, etc.)
  // In production: let it crash and restart (PM2/k8s will restart)
});

// Catch synchronous throws that escaped all try/catch
process.on('uncaughtException', (err) => {
  console.error('Uncaught Exception:', err);
  // ALWAYS exit — process is in undefined state after uncaughtException
  process.exit(1);
});
```

## References

- https://nodejs.org/en/learn/getting-started/nodejs-the-difference-between-development-and-production
- https://12factor.net/config
