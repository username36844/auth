For your stack (**Next.js + Redis + Prisma + stateful sessions**), I would implement **3 different rate limiters**, not one.

Different endpoints need different protection.

---

# 1. Login Rate Limit

Protects against:

```text
Credential stuffing
Brute force attacks
Password guessing
```

Key:

```text
rate:login:ip:{ip}
```

Example:

```text
rate:login:ip:192.168.1.10
```

---

## Redis Sorted Set

Each login attempt stores a timestamp.

```ts
const now = Date.now();

await redis.zAdd(
  `rate:login:ip:${ip}`,
  {
    score: now,
    value: crypto.randomUUID(),
  }
);
```

---

Remove old requests.

Example:

```text
5 minute window
```

```ts
await redis.zRemRangeByScore(
  key,
  0,
  now - 5 * 60 * 1000
);
```

---

Count requests.

```ts
const attempts =
  await redis.zCard(key);
```

---

Rule:

```ts
if (attempts > 5) {
  throw new Error(
    "Too many login attempts"
  );
}
```

Result:

```text
5 login attempts
per 5 minutes
per IP
```

---

# 2. Password Reset Rate Limit

Protects against:

```text
Email spam
Reset abuse
Enumeration attacks
```

Key:

```text
rate:reset:email:test@example.com
```

Rule:

```text
3 reset emails
per hour
```

---

Flow:

```ts
const key =
  `rate:reset:email:${email}`;
```

Store request.

```ts
await redis.zAdd(
  key,
  {
    score: Date.now(),
    value: crypto.randomUUID(),
  }
);
```

Count:

```ts
const count =
  await redis.zCard(key);
```

Block:

```ts
if (count > 3)
```

---

# 3. Email Verification Resend

Protects against:

```text
Email flooding
SMTP abuse
```

Key:

```text
rate:verify:user:123
```

Rule:

```text
5 per day
```

---

# Reusable Function

Create one helper.

```ts
type RateLimitOptions = {
  key: string;
  limit: number;
  windowMs: number;
};
```

---

```ts
export async function rateLimit({
  key,
  limit,
  windowMs,
}: RateLimitOptions) {
  const now = Date.now();

  await redis.zRemRangeByScore(
    key,
    0,
    now - windowMs
  );

  await redis.zAdd(
    key,
    {
      score: now,
      value: crypto.randomUUID(),
    }
  );

  const count =
    await redis.zCard(key);

  await redis.expire(
    key,
    Math.ceil(windowMs / 1000)
  );

  return {
    success: count <= limit,
    remaining:
      Math.max(0, limit - count),
  };
}
```

---

Usage:

```ts
const result =
  await rateLimit({
    key:
      `rate:login:ip:${ip}`,
    limit: 5,
    windowMs: 300000,
  });

if (!result.success) {
  return NextResponse.json(
    {
      error:
        "Too many attempts",
    },
    {
      status: 429,
    }
  );
}
```

---

# Authenticated User Rate Limits

Once logged in, use user ID instead of IP.

Example:

```text
POST /api/projects
```

Key:

```text
rate:create-project:user:123
```

Rule:

```text
20 projects
per hour
```

Much more accurate than IP-based limits.

---

# Combining IP + User Limits

For login:

```text
rate:login:ip:{ip}
```

For authenticated actions:

```text
rate:user:{userId}:action
```

For very sensitive actions:

```text
rate:login:ip:{ip}
rate:login:email:{email}
```

Check both.

---

# Additional Protection: Account Lockout

Store failed login count.

```text
failed_login:user:123
```

Increment only on bad passwords.

```ts
await redis.incr(
  `failed_login:${user.id}`
);
```

TTL:

```ts
await redis.expire(
  key,
  900
);
```

15 minutes.

Rule:

```text
10 failed passwords
```

Then:

```text
Lock account for 15 minutes
```

---

# Redis Keys I'd Use

```text
session:{sid}

user:{userId}

user_sessions:{userId}

verify:{tokenHash}

reset:{tokenHash}

rate:login:ip:{ip}

rate:login:email:{email}

rate:reset:email:{email}

rate:verify:user:{userId}

rate:user:{userId}:{action}

failed_login:{userId}
```

---

For a production SaaS, my typical limits are:

| Action              | Limit                 |
| ------------------- | --------------------- |
| Login attempts      | 5 / 5 min per IP      |
| Login attempts      | 10 / 15 min per email |
| Password reset      | 3 / hour per email    |
| Verification resend | 5 / day per user      |
| Registration        | 5 / hour per IP       |
| Authenticated API   | 60 / min per user     |
| Contact form        | 5 / hour per IP       |

The sorted-set sliding window approach is preferred over simple `INCR` counters because it prevents users from exploiting fixed-window boundaries and gives much smoother rate limiting behavior.
