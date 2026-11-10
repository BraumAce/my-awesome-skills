# Security Review — Code Examples

Detailed code patterns referenced by the security-review skill. Organized by topic.

## 1. Secrets Management

### Bad
```typescript
const apiKey = "sk-proj-xxxxx"  // Hardcoded
const dbPassword = "password123"
```

### Good
```typescript
const apiKey = process.env.OPENAI_API_KEY
if (!apiKey) throw new Error('OPENAI_API_KEY not configured')
```

```python
import os
api_key = os.environ["API_KEY"]  # Fails fast if missing
```

## 2. Input Validation

### Schema Validation (TypeScript)
```typescript
import { z } from 'zod'

const CreateUserSchema = z.object({
  email: z.string().email(),
  name: z.string().min(1).max(100),
  age: z.number().int().min(0).max(150)
})

export async function createUser(input: unknown) {
  const validated = CreateUserSchema.parse(input)
  return await db.users.create(validated)
}
```

### Schema Validation (Python)
```python
from pydantic import BaseModel, EmailStr

class CreateUser(BaseModel):
    email: EmailStr
    name: str = Field(min_length=1, max_length=100)
    age: int = Field(ge=0, le=150)
```

### File Upload
```typescript
function validateUpload(file: File) {
  if (file.size > 5 * 1024 * 1024) throw new Error('Max 5MB')
  const allowed = ['image/jpeg', 'image/png', 'image/gif']
  if (!allowed.includes(file.type)) throw new Error('Invalid type')
}
```

## 3. SQL Injection Prevention

### Bad
```typescript
const query = `SELECT * FROM users WHERE email = '${userEmail}'`
```

### Good
```typescript
// ORM / Query builder
const { data } = await supabase.from('users').select('*').eq('email', userEmail)

// Raw SQL with parameters
await db.query('SELECT * FROM users WHERE email = $1', [userEmail])
```

```python
# Python
cursor.execute("SELECT * FROM users WHERE email = %s", (email,))
```

## 4. XSS Prevention

### Sanitize User HTML
```typescript
import DOMPurify from 'isomorphic-dompurify'

const clean = DOMPurify.sanitize(userHtml, {
  ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'p'],
  ALLOWED_ATTR: []
})
```

### Content Security Policy
```typescript
// next.config.js
headers: [{ key: 'Content-Security-Policy', value: "default-src 'self'; script-src 'self'" }]
```

## 5. Authentication & Authorization

### Token Storage
```typescript
// Bad: localStorage (XSS vulnerable)
localStorage.setItem('token', token)

// Good: httpOnly cookie
res.setHeader('Set-Cookie', `token=${token}; HttpOnly; Secure; SameSite=Strict; Max-Age=3600`)
```

### Authorization Check
```typescript
export async function deleteUser(userId: string, requesterId: string) {
  const requester = await db.users.findUnique({ where: { id: requesterId } })
  if (requester.role !== 'admin') {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 403 })
  }
  await db.users.delete({ where: { id: userId } })
}
```

### Row Level Security (Supabase)
```sql
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
CREATE POLICY "Users view own data" ON users FOR SELECT USING (auth.uid() = id);
CREATE POLICY "Users update own data" ON users FOR UPDATE USING (auth.uid() = id);
```

## 6. CSRF Protection

```typescript
export async function POST(request: Request) {
  const token = request.headers.get('X-CSRF-Token')
  if (!csrf.verify(token)) {
    return NextResponse.json({ error: 'Invalid CSRF token' }, { status: 403 })
  }
}
```

## 7. Rate Limiting

```typescript
import rateLimit from 'express-rate-limit'

app.use('/api/', rateLimit({ windowMs: 15 * 60 * 1000, max: 100 }))
app.use('/api/search', rateLimit({ windowMs: 60 * 1000, max: 10 }))
app.use('/api/auth', rateLimit({ windowMs: 15 * 60 * 1000, max: 5 }))
```

## 8. Error Handling

### Bad
```typescript
catch (error) {
  return NextResponse.json({ error: error.message, stack: error.stack }, { status: 500 })
}
```

### Good
```typescript
catch (error) {
  console.error('Internal error:', error)  // Server-side only
  return NextResponse.json({ error: 'An error occurred.' }, { status: 500 })
}
```

## 9. Security Testing Examples

```typescript
test('requires authentication', async () => {
  const res = await fetch('/api/protected')
  expect(res.status).toBe(401)
})

test('requires admin role', async () => {
  const res = await fetch('/api/admin', { headers: { Authorization: `Bearer ${userToken}` } })
  expect(res.status).toBe(403)
})

test('rejects invalid input', async () => {
  const res = await fetch('/api/users', { method: 'POST', body: JSON.stringify({ email: 'bad' }) })
  expect(res.status).toBe(400)
})

test('enforces rate limits', async () => {
  const responses = await Promise.all(Array(101).fill(null).map(() => fetch('/api/endpoint')))
  expect(responses.filter(r => r.status === 429).length).toBeGreaterThan(0)
})
```
