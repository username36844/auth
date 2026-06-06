This is the architecture I would build today for a serious Next.js application.

---

# High-Level Architecture

```text
Browser
│
├── HttpOnly Secure Cookie
│       sid=session_id
│
▼
Next.js Middleware
│
▼
Redis Session
│
├── user_id
├── auth_version
├── created_at
└── last_seen
│
▼
Redis User Cache
│
├── status
├── role
├── auth_version
└── email_verified
│
▼
PostgreSQL (Prisma)
```

Database remains the source of truth.

Redis is the authentication cache.

Cookie only identifies the session.

---

# Prisma Schema

```prisma
model User {
  id              String   @id @default(cuid())
  email           String   @unique
  passwordHash    String

  role            UserRole @default(USER)

  status          UserStatus @default(ACTIVE)

  emailVerified   Boolean @default(false)

  authVersion     Int @default(1)

  createdAt       DateTime @default(now())
  updatedAt       DateTime @updatedAt
}

enum UserRole {
  USER
  ADMIN
}

enum UserStatus {
  ACTIVE
  SUSPENDED
  BANNED
}
```

---

# Redis Keys

## Session

```text
session:e4e88f6f...
```

Hash:

```text
user_id=123
auth_version=5
created_at=171000000
last_seen=171000200
```

TTL:

```text
7 days
```

---

## User Cache

```text
user:123
```

Hash:

```text
status=ACTIVE
role=USER
email_verified=true
auth_version=5
```

TTL:

```text
24 hours
```

or no expiration.

---

## User Sessions

```text
user_sessions:123
```

Set:

```text
sid1
sid2
sid3
```

Useful for:

```text
show active devices
logout current device
logout specific device
```

---

# Login Flow

User submits:

```tsx
email
password
```

Validated with Zod.

```ts
const LoginSchema = z.object({
  email: z.email(),
  password: z.string().min(8),
});
```

React Hook Form:

```ts
const form = useForm<LoginInput>({
  resolver: zodResolver(LoginSchema),
});
```

---

# Login Route

Find user:

```ts
const user = await prisma.user.findUnique({
  where: {
    email,
  },
});
```

Verify password:

```ts
const valid = await verify(
  user.passwordHash,
  password
);
```

---

Create Session ID

```ts
import crypto from "crypto";

const sid =
  crypto.randomBytes(32).toString("hex");
```

---

Create Redis Session

```ts
await redis.hSet(
  `session:${sid}`,
  {
    user_id: user.id,
    auth_version: user.authVersion,
    created_at: Date.now(),
    last_seen: Date.now(),
  }
);

await redis.expire(
  `session:${sid}`,
  60 * 60 * 24 * 7
);
```

---

Create User Cache

```ts
await redis.hSet(
  `user:${user.id}`,
  {
    status: user.status,
    role: user.role,
    email_verified:
      user.emailVerified ? "1" : "0",
    auth_version:
      user.authVersion,
  }
);
```

---

Track Session

```ts
await redis.sAdd(
  `user_sessions:${user.id}`,
  sid
);
```

---

Set Cookie

```ts
cookies().set(
  "sid",
  sid,
  {
    httpOnly: true,
    secure: true,
    sameSite: "strict",
    path: "/",
    maxAge: 60 * 60 * 24 * 7,
  }
);
```

Cookie contains ONLY:

```text
sid=e4e88f...
```

Nothing else.

---

# Middleware Flow

Request arrives.

Read cookie:

```ts
const sid =
  request.cookies.get("sid")?.value;
```

---

Lookup Session

```ts
const session =
  await redis.hGetAll(
    `session:${sid}`
  );
```

Checks:

```ts
if (!session)
```

Reject.

---

Lookup User Cache

```ts
const user =
  await redis.hGetAll(
    `user:${session.user_id}`
  );
```

---

# Check 1

Session Exists

```ts
if (!session)
```

401

---

# Check 2

User Exists

```ts
if (!user)
```

401

---

# Check 3

Auth Version

```ts
if (
  Number(session.auth_version) !==
  Number(user.auth_version)
)
```

401

Delete session.

Clear cookie.

---

# Check 4

User Status

```ts
if (
  user.status !== "ACTIVE"
)
```

403

---

# Check 5

Email Verification

For protected routes:

```ts
if (
  user.email_verified !== "1"
)
```

403

---

# Check 6

Role

Admin route:

```ts
if (
  user.role !== "ADMIN"
)
```

403

---

After checks:

```ts
request.user = {
  id: session.user_id,
  role: user.role,
};
```

Continue.

---

# Password Change Flow

User changes password.

Update DB:

```ts
await prisma.user.update({
  where: {
    id: userId,
  },
  data: {
    passwordHash: hash,
    authVersion: {
      increment: 1,
    },
  },
});
```

---

Update Redis Cache

```ts
await redis.hSet(
  `user:${userId}`,
  {
    auth_version:
      currentVersion + 1,
  }
);
```

Done.

---

Current Sessions

```text
session auth_version = 5
```

User Cache

```text
auth_version = 6
```

Next request:

```ts
5 !== 6
```

All sessions invalid.

---

# Account Suspension

Admin clicks:

```text
Suspend User
```

Update DB:

```ts
await prisma.user.update({
  where: {
    id: userId,
  },
  data: {
    status: "SUSPENDED",
    authVersion: {
      increment: 1,
    },
  },
});
```

Update Cache:

```ts
await redis.hSet(
  `user:${userId}`,
  {
    status: "SUSPENDED",
    auth_version:
      newVersion,
  }
);
```

Every session immediately fails.

---

# Logout

Current session only.

```ts
await redis.del(
  `session:${sid}`
);
```

Remove from set:

```ts
await redis.sRem(
  `user_sessions:${userId}`,
  sid
);
```

Clear cookie.

---

# Logout All Devices

No session iteration required.

Update DB:

```ts
authVersion += 1
```

Update Redis Cache:

```ts
auth_version += 1
```

Done.

All sessions become invalid.

---

# Email Verification

Database:

```ts
emailVerified = true
```

Update Cache:

```ts
await redis.hSet(
  `user:${userId}`,
  {
    email_verified: "1",
  }
);
```

No authVersion bump needed.

---

# Rate Limiting

Login endpoint:

```text
rate:login:ip
```

Sorted Set.

Request:

```ts
const now = Date.now();

await redis.zAdd(
  key,
  {
    score: now,
    value: crypto.randomUUID(),
  }
);
```

Remove old entries:

```ts
await redis.zRemRangeByScore(
  key,
  0,
  now - 5 * 60 * 1000
);
```

Count:

```ts
const count =
  await redis.zCard(key);
```

Rule:

```ts
count > 5
```

Reject login.

---

# Security Benefits

### Cookie Theft

Attacker gets:

```text
sid
```

But if:

```text
password changed
account suspended
logout all devices
```

authVersion changes.

Session dies.

---

### Role Change

```text
USER → ADMIN
```

or

```text
ADMIN → USER
```

Increment:

```text
authVersion
```

Old permissions disappear instantly.

---

### Database Queries

Normal request:

```text
Cookie
↓
Redis Session
↓
Redis User Cache
↓
Authorized
```

No database hit.

Only Redis.

---

# Final Production Flow

```text
Cookie
│
└── sid

Redis
│
├── session:{sid}
│     ├── user_id
│     └── auth_version
│
└── user:{user_id}
      ├── status
      ├── role
      ├── email_verified
      └── auth_version

PostgreSQL
│
└── Source of Truth
```

Checks performed on every authenticated request:

1. Cookie exists.
2. Session exists in Redis.
3. User cache exists in Redis.
4. Session auth_version === User auth_version.
5. User status is ACTIVE.
6. Email verified if required.
7. Role permissions pass.
8. Session TTL valid.

If any check fails:

```text
Delete session
Clear cookie
Return 401/403
```

This gives you stateful sessions, instant revocation, logout-all-devices, role synchronization, account suspension, and near-zero database load on authenticated requests.



One thing I would add

Instead of:

Redis only

I'd store session metadata in PostgreSQL too:

model Session {
  id           String   @id @default(cuid())
  userId       String

  sessionHash  String   @unique

  createdAt    DateTime @default(now())
  expiresAt    DateTime

  revokedAt    DateTime?

  user         User @relation(fields: [userId], references: [id])
}

Then:

Redis
→ Fast auth path

PostgreSQL
→ Audit trail
→ Device management
→ Session history
→ Security investigations
