# nohumanallowed

> Anti-human verification for the agentic web. Proof-of-work captcha for AI agents.

[![npm version](https://img.shields.io/npm/v/nohumanallowed.svg)](https://www.npmjs.com/package/nohumanallowed)
[![License](https://img.shields.io/badge/license-Apache%202.0-blue.svg)](LICENSE)

**Powered by [Shellborn Collective](https://shellborn.io)** 🦞

## What is this?

A computational proof-of-work challenge system designed to verify AI agents, not humans. While traditional CAPTCHAs block bots, No Human Allowed blocks humans—ensuring only automated agents can pass.

**Use cases:**
- Agent-only NFT mints
- Bot-exclusive API endpoints
- Agentic service verification
- Rate limiting with computational cost

## Installation

```bash
npm install nohumanallowed
```

Or use directly with npx (no install needed):

```bash
npx nohumanallowed --help
```

## Quick Start

### CLI Usage

```bash
# Create a challenge (difficulty 4 = 4 leading zeros)
npx nohumanallowed challenge --difficulty 4

# Solve a challenge
npx nohumanallowed solve --prefix <prefix> --target 0000

# Verify a solution
npx nohumanallowed verify --prefix <prefix> --nonce <nonce> --target 0000 --expires <expiresAt>
```

### Programmatic Usage

```typescript
import { createChallenge, verifyChallenge } from 'nohumanallowed'
import { solveChallenge } from 'nohumanallowed/client'

// Server: Create a challenge
const challenge = createChallenge({
  difficulty: 4,      // 4 leading zeros (~40ms solve time)
  expiresIn: 60,      // 60 seconds to solve
  secret: process.env.NHA_SECRET,  // Sign for tamper protection
})

// Client/Agent: Solve the challenge
const solution = await solveChallenge({
  prefix: challenge.prefix,
  target: challenge.target,
})

// Server: Verify the solution
const result = verifyChallenge({
  prefix: challenge.prefix,
  nonce: solution.nonce,
  target: challenge.target,
  expiresAt: challenge.expiresAt,
  signature: challenge.signature,
  secret: process.env.NHA_SECRET,
})

if (result.valid) {
  // Agent verified! Grant access
}
```

## API Reference

### Server-Side (Main Export)

```typescript
import { createChallenge, verifyChallenge, generateToken, verifyToken } from 'nohumanallowed'
```

#### `createChallenge(options?)`

Creates a new verification challenge.

```typescript
interface ChallengeOptions {
  difficulty?: number    // Leading zeros required (1-6). Default: 4
  expiresIn?: number     // Seconds until expiry. Default: 60
  metadata?: object      // Optional data to embed (included in signature)
  secret?: string        // Signs challenge for tamper protection (recommended)
}

interface Challenge {
  id: string             // Unique challenge ID (nha_...)
  prefix: string         // Random prefix to hash with nonce
  target: string         // Target pattern (e.g., "0000")
  expiresAt: number      // Unix timestamp when challenge expires
  metadata?: object      // Embedded metadata (if provided)
  signature?: string     // HMAC signature (if secret provided)
}
```

**Difficulty guide:**

| Difficulty | Target | Avg. Solve Time | Use Case |
|------------|--------|-----------------|----------|
| 1 | `0` | ~0.1ms | Testing |
| 2 | `00` | ~1ms | Lightweight |
| 3 | `000` | ~10ms | Standard |
| 4 | `0000` | ~40ms | **Recommended** |
| 5 | `00000` | ~600ms | High security |
| 6 | `000000` | ~10s | Maximum |

#### `verifyChallenge(options)`

Verifies a submitted solution.

```typescript
interface VerifyOptions {
  prefix: string           // Original challenge prefix
  nonce: string            // Submitted nonce
  target: string           // Target pattern
  expiresAt: number        // Challenge expiration timestamp
  signature?: string       // Challenge signature (if signed)
  secret?: string          // Your secret (required if signature present)
  requireSignature?: bool  // Fail if no valid signature (default: false)
  metadata?: object        // Metadata (must match if signed)
}

interface VerifyResult {
  valid: boolean
  reason?: 'expired' | 'invalid_hash' | 'invalid_signature' | 
           'missing_signature' | 'missing_secret' | 'invalid_input' |
           'invalid_target' | 'invalid_expiry'
  hash?: string            // The computed hash (for debugging)
}
```

#### `generateToken(challengeId, secret?)`

Generates a verification token after successful verification. Use this to grant access without re-verifying.

```typescript
const token = generateToken(challenge.id, process.env.NHA_SECRET)
// Returns: base64payload.signature (if secret) or just base64payload
```

#### `verifyToken(token, secret?, maxAgeMs?, options?)`

Verifies a previously generated token.

```typescript
const result = verifyToken(token, process.env.NHA_SECRET, 300000) // 5 min max age
// Returns: { valid: boolean, challengeId?: string }
```

### Client-Side (Agent Solver)

```typescript
import { solveChallenge, estimateSolveTime } from 'nohumanallowed/client'
```

#### `solveChallenge(options)`

Brute-forces a solution to the challenge.

```typescript
interface SolveOptions {
  prefix: string              // Challenge prefix
  target: string              // Target pattern
  maxIterations?: number      // Give up after N tries (default: 10M)
  onProgress?: (iterations, hashRate) => void  // Progress callback
  progressInterval?: number   // Callback frequency (default: 100K)
}

interface SolveResult {
  found: boolean
  nonce?: string       // The solution (if found)
  iterations: number   // Total attempts
  timeMs: number       // Time taken
  hashRate: number     // Hashes per second
  hash?: string        // Resulting hash (if found)
}
```

#### `estimateSolveTime(difficulty)`

Estimates solve time for a given difficulty.

```typescript
const estimate = estimateSolveTime(4)
// Returns: { averageMs: 40, description: "Nearly instant" }
```

## Hosted API

Free hosted API at `nohumanallowed.com` (rate limited):

```bash
# Get a challenge
curl -L https://nohumanallowed.com/api/challenge

# Get a challenge with custom difficulty
curl -L "https://nohumanallowed.com/api/challenge?difficulty=5"

# Verify a solution
curl -L -X POST https://nohumanallowed.com/api/verify \
  -H "Content-Type: application/json" \
  -d '{
    "prefix": "abc123...",
    "nonce": "12345",
    "target": "0000",
    "expiresAt": 1707350400,
    "signature": "..."
  }'
```

**Response formats:**

```typescript
// Challenge response
{
  "id": "nha_abc123...",
  "prefix": "f7e2a1b3...",
  "target": "0000",
  "expiresAt": 1707350400,
  "signature": "8a3f..."
}

// Verify response (success)
{
  "valid": true,
  "token": "base64...",
  "hash": "0000abc..."
}

// Verify response (failure)
{
  "valid": false,
  "reason": "expired"
}
```

## Security Considerations

> ⚠️ **IMPORTANT**: Always use a `secret` in production. Without a secret, challenges are unsigned and can be forged by attackers.

### Always Use a Secret in Production

Without a secret, challenges can be forged by malicious actors:

```typescript
// ❌ INSECURE - anyone can create valid challenges
const challenge = createChallenge({ difficulty: 4 })

// ✅ SECURE - challenges are signed and tamper-proof
const challenge = createChallenge({
  difficulty: 4,
  secret: process.env.NHA_SECRET,  // Use a strong secret (32+ chars)
})

// ✅ MOST SECURE - require signature verification
const result = verifyChallenge({
  ...solution,
  secret: process.env.NHA_SECRET,
  requireSignature: true,  // Fails if no valid signature
})
```

### Replay Protection

By design, nohumanallowed is **stateless** — the same nonce can be submitted multiple times until the challenge expires. For high-security applications requiring replay protection:

```typescript
// Server-side nonce tracking example
const usedNonces = new Set<string>()  // Use Redis in production

app.post('/api/protected', (req, res) => {
  const { prefix, nonce, target, expiresAt, signature } = req.body
  
  // 1. Check for replay
  const nonceKey = `${prefix}:${nonce}`
  if (usedNonces.has(nonceKey)) {
    return res.status(403).json({ error: 'Nonce already used' })
  }
  
  // 2. Verify the challenge
  const result = verifyChallenge({
    prefix, nonce, target, expiresAt, signature,
    secret: process.env.NHA_SECRET,
    requireSignature: true,
  })
  
  if (!result.valid) {
    return res.status(403).json({ error: result.reason })
  }
  
  // 3. Mark nonce as used (with TTL matching challenge expiry)
  usedNonces.add(nonceKey)
  setTimeout(() => usedNonces.delete(nonceKey), 120000)  // 2 min TTL
  
  // Proceed with protected action
  res.json({ success: true })
})
```

### Fail-Closed Behavior

If a signature is present but no secret is provided to verify, verification **fails**:

```typescript
// Challenge was signed
const challenge = createChallenge({ secret: 'mysecret' })

// Verification without secret = FAILS (not bypassed)
verifyChallenge({
  ...solution,
  signature: challenge.signature,
  // secret: undefined  ← Missing!
})
// Returns: { valid: false, reason: 'missing_secret' }
```

### Token Expiration

Always set appropriate expiration times:

```typescript
// Short-lived challenges (recommended)
createChallenge({ expiresIn: 60 })  // 1 minute

// Short-lived tokens
verifyToken(token, secret, 300000)  // 5 minute max age
```

### Security Checklist

For production deployments:

- [ ] **Always provide a secret** to `createChallenge()` and `verifyChallenge()`
- [ ] **Use `requireSignature: true`** in `verifyChallenge()` for fail-closed behavior
- [ ] **Implement replay protection** if needed (see example above)
- [ ] **Use short expiration times** (60s for challenges, 5min for tokens)
- [ ] **Store secrets securely** (environment variables, secrets manager)

## Integration Examples

### Express.js Middleware

```typescript
import express from 'express'
import { createChallenge, verifyChallenge } from 'nohumanallowed'

const app = express()
const SECRET = process.env.NHA_SECRET

app.get('/api/challenge', (req, res) => {
  const challenge = createChallenge({
    difficulty: 4,
    secret: SECRET,
    metadata: { action: 'mint', userId: req.user?.id }
  })
  res.json(challenge)
})

app.post('/api/protected', (req, res) => {
  const { prefix, nonce, target, expiresAt, signature, metadata } = req.body
  
  const result = verifyChallenge({
    prefix, nonce, target, expiresAt, signature, metadata,
    secret: SECRET,
    requireSignature: true,
  })
  
  if (!result.valid) {
    return res.status(403).json({ error: 'Verification failed', reason: result.reason })
  }
  
  // Proceed with protected action
  res.json({ success: true })
})
```

### Agent Integration (OpenClaw Skill)

```bash
# Fetch challenge
CHALLENGE=$(curl -sL https://example.com/api/challenge)
PREFIX=$(echo $CHALLENGE | jq -r .prefix)
TARGET=$(echo $CHALLENGE | jq -r .target)

# Solve it
NONCE=$(npx nohumanallowed solve --prefix "$PREFIX" --target "$TARGET")

# Submit solution
curl -X POST https://example.com/api/protected \
  -H "Content-Type: application/json" \
  -d "$CHALLENGE" \
  --data-urlencode "nonce=$NONCE"
```

## How It Works

1. **Challenge Creation**: Server generates a random prefix and target pattern (leading zeros)
2. **Proof of Work**: Client finds a nonce where `SHA256(prefix + nonce)` starts with target
3. **Verification**: Server verifies the hash matches and challenge hasn't expired
4. **Token (optional)**: Server issues a short-lived token for subsequent requests

The computational cost scales exponentially with difficulty—each additional zero roughly 16x the work.

## Why Proof-of-Work?

Unlike CAPTCHAs that test human perception, proof-of-work:

- ✅ **Favors machines** — Agents solve faster than humans can click
- ✅ **Cryptographically verifiable** — No AI image recognition needed
- ✅ **Adjustable difficulty** — Scale from trivial to expensive
- ✅ **No third parties** — Self-contained, no external services
- ✅ **Works offline** — No network calls during solving

## License

Apache 2.0 — See [LICENSE](LICENSE) for details.

**Attribution required**: Include "Powered by Shellborn" or link to shellborn.io

---

Built by [Shellborn Collective](https://shellborn.io) 🦞

Website: [nohumanallowed.com](https://nohumanallowed.com)
