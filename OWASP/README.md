# 🔐 OWASP Top 10 – 2025 with TypeScript Examples

The **OWASP Top 10** is a standard awareness document for developers and web application security. Here’s the **2025 edition**, explained with practical **TypeScript** code snippets.

---

## 1. Broken Access Control

**Risk:** Attackers can access or modify unauthorized data.

### Example 1 – Insecure Direct Object Reference (IDOR)

```ts
// ❌ Insecure
app.get('/orders/:id', async (req, res) => {
  const order = await getOrderById(req.params.id);
  res.json(order); // No user ownership check
});

// ✅ Secure
app.get('/orders/:id', async (req, res) => {
  const order = await getOrderById(req.params.id);
  if (order.userId !== req.user.id) return res.status(403).send('Forbidden');
  res.json(order);
});
```

### Example 2 – Missing Role Check

```ts
// ❌
app.post('/admin/deleteUser', deleteUser);

// ✅
app.post('/admin/deleteUser', (req, res) => {
  if (req.user.role !== 'admin') return res.status(403).send('Forbidden');
  deleteUser(req, res);
});
```

---

## 2. Cryptographic Failures

**Risk:** Weak encryption, no encryption, or key exposure.

### Example 1 – Password Storage

```ts
// ❌
await db.insert({ username, password });

// ✅
const hashed = await bcrypt.hash(password, 12);
await db.insert({ username, password: hashed });
```

### Example 2 – Hardcoded Secrets

```ts
// ❌
const jwtSecret = 'my-secret';

// ✅
const jwtSecret = process.env.JWT_SECRET!;
```

---

## 3. Injection

**Risk:** Unsanitized input interpreted as code.

### Example 1 – SQL Injection

```ts
// ❌
db.query(`SELECT * FROM users WHERE email = '${req.body.email}'`);

// ✅
db.query('SELECT * FROM users WHERE email = $1', [req.body.email]);
```

### Example 2 – NoSQL Injection

```ts
// ❌
User.find({ username: req.body.username });

// ✅
User.find({ username: sanitize(req.body.username) });
```

---

## 4. Insecure Design

**Risk:** Lack of secure defaults, validations, and secure design thinking.

### Example 1 – No Rate Limiting

```ts
// ❌
app.post('/login', loginHandler);

// ✅
app.post('/login', rateLimiter, loginHandler);
```

### Example 2 – No Input Validation

```ts
// ✅ with Zod
const schema = z.object({ email: z.string().email(), password: z.string().min(6) });
const result = schema.safeParse(req.body);
if (!result.success) return res.status(400).json(result.error);
```

---

## 5. Security Misconfiguration

**Risk:** Misconfigured headers, unnecessary services, detailed errors.

### Example 1 – Stack Trace Exposure

```ts
// ❌
app.use((err, req, res) => res.status(500).send(err.stack));

// ✅
app.use((err, req, res) => {
  res.status(500).send('Server error');
  logger.error(err.stack);
});
```

### Example 2 – Open Admin Panel

```ts
// ✅
app.use('/admin', authMiddleware, express.static('admin'));
```

---

## 6. Vulnerable and Outdated Components

**Risk:** Using libraries with known CVEs.

### Example 1 – Audit dependencies

```bash
npm audit fix
```

### Example 2 – Use Tools

* Snyk
* Dependabot

---

## 7. Identification and Authentication Failures

**Risk:** Poor login logic, session mismanagement.

### Example 1 – JWT Expiry

```ts
// ❌
jwt.sign(payload, secret);

// ✅
jwt.sign(payload, secret, { expiresIn: '1h' });
```

### Example 2 – No Logout Invalidation

```ts
// ✅
app.post('/logout', (req, res) => {
  req.session.destroy(() => res.send('Logged out'));
});
```

---

## 8. Software and Data Integrity Failures

**Risk:** Unauthorized code execution from updates, plugins.

### Example 1 – Subresource Integrity

```html
<script src="https://cdn.com/lib.js"
  integrity="sha384-..." crossorigin="anonymous"></script>
```

### Example 2 – CI/CD Protections

```yaml
on:
  pull_request:
    branches: [main]
```

---

## 9. Security Logging and Monitoring Failures

**Risk:** No detection of breaches or attacks.

### Example 1 – Login Logging

```ts
if (!validUser) {
  logger.warn(`Failed login for ${req.body.username}`);
}
```

### Example 2 – Audit Trail

```ts
logger.info(`User ${req.user.id} updated profile`);
```

---

## 10. Server-Side Request Forgery (SSRF)

**Risk:** Server is tricked into accessing internal resources.

### Example 1 – Whitelisted URLs

```ts
const allowed = ['https://api.example.com'];
if (!allowed.includes(req.query.url)) return res.status(400).send('Blocked');
axios.get(req.query.url);
```

### Example 2 – DNS Validation

```ts
const hostname = new URL(req.query.url).hostname;
const ip = (await dns.lookup(hostname)).address;
if (ip.startsWith('169.254') || ip.startsWith('127.') || ip.startsWith('10.')) {
  throw new Error('SSRF blocked');
}
```

---

## 📌 Summary

Always:

* Apply **principle of least privilege**
* Use **automated scanning tools**
* Perform **secure code reviews**
* Build with **security by design**

Stay safe and ship secure software! 🚀

---

*Created by a passionate backend engineer using TypeScript & Node.js to build secure apps.*
