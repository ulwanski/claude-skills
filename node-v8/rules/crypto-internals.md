---
name: crypto-internals
description: OpenSSL integration, thread-pool vs main-thread crypto, security patterns
metadata:
  tags: crypto, openssl, security, performance
---

# Node.js Crypto Internals

## Thread pool vs. main thread

```javascript
// Uses THREAD POOL (async, non-blocking):
crypto.pbkdf2(password, salt, 100000, 64, 'sha512', callback);
crypto.scrypt(password, salt, keylen, callback);
crypto.randomBytes(256, callback);
crypto.generateKeyPair('rsa', options, callback);

// Runs on MAIN THREAD (sync — avoid in hot paths):
crypto.createHash('sha256').update(data).digest();
crypto.createHmac('sha256', key).update(data).digest();
crypto.createCipheriv(algorithm, key, iv);
```

## Timing attacks — always use timingSafeEqual

```javascript
// BAD: vulnerable to timing attack
if (expected === actual) { ... }

// GOOD: constant-time comparison
if (!crypto.timingSafeEqual(Buffer.from(expected), Buffer.from(actual))) {
  throw new Error('Invalid signature');
}
```

## IV must be unique per message (AES-GCM, AES-CBC)

```javascript
// BAD: reusing IV
const iv = crypto.randomBytes(12);
messages.forEach(msg => encrypt(msg, key, iv)); // SECURITY BUG

// GOOD: fresh IV per message
messages.forEach(msg => {
  const iv = crypto.randomBytes(12); // unique each time
  encrypt(msg, key, iv);
});
```

## Preferred: AES-256-GCM (authenticated encryption)

```javascript
function encrypt(plaintext, key) {
  const iv = crypto.randomBytes(12);
  const cipher = crypto.createCipheriv('aes-256-gcm', key, iv);
  const ciphertext = Buffer.concat([
    cipher.update(plaintext),
    cipher.final(),
  ]);
  const authTag = cipher.getAuthTag();
  return { iv, ciphertext, authTag };
}

function decrypt({ iv, ciphertext, authTag }, key) {
  const decipher = crypto.createDecipheriv('aes-256-gcm', key, iv);
  decipher.setAuthTag(authTag);
  return Buffer.concat([decipher.update(ciphertext), decipher.final()]);
}
```

## Clear sensitive buffers after use

```javascript
function deriveKey(password, salt) {
  const key = crypto.scryptSync(password, salt, 32);
  try {
    return doSomething(key);
  } finally {
    key.fill(0); // overwrite in memory
  }
}
```

## Password hashing: prefer scrypt over pbkdf2

```javascript
// scrypt is memory-hard (harder to brute-force with GPUs/ASICs)
const key = await promisify(crypto.scrypt)(password, salt, 64, {
  N: 16384, // CPU/memory cost
  r: 8,     // block size
  p: 1,     // parallelism
});
```

## Avoid blocking the event loop with sync crypto

```javascript
// BAD in server hot path
app.post('/login', (req, res) => {
  const hash = crypto.pbkdf2Sync(password, salt, 100000, 64, 'sha512');
  // server blocked for ~100ms
});

// GOOD
app.post('/login', async (req, res) => {
  const hash = await promisify(crypto.pbkdf2)(password, salt, 100000, 64, 'sha512');
});
```

## References

- https://nodejs.org/api/crypto.html
- https://developer.mozilla.org/en-US/docs/Web/API/Web_Crypto_API
