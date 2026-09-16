# OWASP Top 10 Interview Questions & Answers

A curated list of **OWASP Top 10 (2021 edition)** interview questions covering the most critical web application security risks, with practical **TypeScript** code examples of insecure vs. secure implementations. Check [owasp.org](https://owasp.org/Top10/) for future updates to the list.

---

## 1. **What is Broken Access Control and how do you prevent it?**

**Answer:** Broken Access Control happens when an application doesn't properly enforce restrictions on what authenticated users are allowed to do, letting attackers access or modify data they don't own. Prevent it by checking resource ownership and role on every request, not just at the UI layer.

```ts
// ❌ No ownership check
app.get("/orders/:id", async (req, res) => {
  const order = await getOrderById(req.params.id);
  res.json(order);
});

// ✅ Verify the requester owns the resource
app.get("/orders/:id", async (req, res) => {
  const order = await getOrderById(req.params.id);
  if (order.userId !== req.user.id) return res.status(403).send("Forbidden");
  res.json(order);
});

// ✅ Also check role for privileged routes
app.post("/admin/deleteUser", (req, res) => {
  if (req.user.role !== "admin") return res.status(403).send("Forbidden");
  deleteUser(req, res);
});
```

---

## 2. **What are Cryptographic Failures and how do you avoid them?**

**Answer:** Cryptographic Failures (formerly "Sensitive Data Exposure") cover weak or missing encryption, and secrets that leak because they're hardcoded or stored in plaintext. Always hash passwords with a slow algorithm like bcrypt and load secrets from environment variables or a secrets manager, never from source code.

```ts
// ❌ Plaintext password, hardcoded secret
await db.insert({ username, password });
const jwtSecret = "my-secret";

// ✅ Hash passwords, load secrets from env
const hashed = await bcrypt.hash(password, 12);
await db.insert({ username, password: hashed });
const jwtSecret = process.env.JWT_SECRET!;
```

---

## 3. **What is Injection and how do you prevent it?**

**Answer:** Injection flaws occur when untrusted input is interpreted as part of a command or query — SQL, NoSQL, OS commands, etc. Prevent it with parameterized queries and by never trusting user input to shape a query's structure.

```ts
// ❌ SQL injection via string interpolation
db.query(`SELECT * FROM users WHERE email = '${req.body.email}'`);

// ✅ Parameterized query
db.query("SELECT * FROM users WHERE email = $1", [req.body.email]);

// ❌ NoSQL injection — operators can be passed as objects
User.find({ username: req.body.username });

// ✅ Sanitize/validate input first
User.find({ username: sanitize(req.body.username) });
```

---

## 4. **What is Insecure Design and how does it differ from an implementation bug?**

**Answer:** Insecure Design is a category of risk stemming from missing or ineffective security controls at the *design* stage — a flaw in the architecture itself, not just a coding mistake. Even a flawless implementation of an insecure design is still vulnerable. Mitigate it with rate limiting, threat modeling, and input validation built in from the start.

```ts
// ✅ Rate-limit sensitive endpoints
app.post("/login", rateLimiter, loginHandler);

// ✅ Validate input shape with a schema (Zod)
const schema = z.object({
  email: z.string().email(),
  password: z.string().min(6),
});
const result = schema.safeParse(req.body);
if (!result.success) return res.status(400).json(result.error);
```

---

## 5. **What is Security Misconfiguration?**

**Answer:** Security Misconfiguration covers missing hardening across any layer of the stack — verbose error output, unnecessary services or routes left exposed, default credentials, or misconfigured headers. Fix it with generic error responses, server-side logging, and auth middleware on every sensitive route.

```ts
// ❌ Leaks stack trace to the client
app.use((err, req, res) => res.status(500).send(err.stack));

// ✅ Generic response, log details server-side
app.use((err, req, res) => {
  res.status(500).send("Server error");
  logger.error(err.stack);
});

// ✅ Protect admin/static routes with auth middleware
app.use("/admin", authMiddleware, express.static("admin"));
```

---

## 6. **How do you handle Vulnerable and Outdated Components?**

**Answer:** This risk comes from using libraries or frameworks with known CVEs. Mitigate it by auditing dependencies regularly and wiring automated scanning into CI so vulnerable packages are caught before they ship.

```bash
npm audit fix
```

Also use automated tools like **Snyk** or **Dependabot** to catch vulnerable dependencies continuously.

---

## 7. **What are Identification and Authentication Failures?**

**Answer:** This category covers weaknesses in login logic and session management — tokens that never expire, sessions that aren't invalidated on logout, weak password policies, or missing brute-force protection.

```ts
// ❌ Token never expires
jwt.sign(payload, secret);

// ✅ Set an expiry
jwt.sign(payload, secret, { expiresIn: "1h" });

// ✅ Invalidate session on logout
app.post("/logout", (req, res) => {
  req.session.destroy(() => res.send("Logged out"));
});
```

---

## 8. **What are Software and Data Integrity Failures?**

**Answer:** This risk covers code and infrastructure that don't verify integrity — unsigned software updates, plugins from untrusted sources, or CI/CD pipelines without adequate access control, allowing malicious code to be introduced. Use Subresource Integrity for third-party scripts and require review before merging into protected branches.

```html
<!-- ✅ Subresource Integrity for third-party scripts -->
<script
  src="https://cdn.com/lib.js"
  integrity="sha384-..."
  crossorigin="anonymous"
></script>
```

```yaml
# ✅ Require PR review before merging into a protected branch
on:
  pull_request:
    branches: [main]
```

---

## 9. **What are Security Logging and Monitoring Failures?**

**Answer:** Without adequate logging and monitoring, breaches go undetected — sometimes for months — because there's no record of suspicious activity to alert on. Log authentication events and sensitive actions, and feed them into a monitoring/alerting pipeline.

```ts
// ✅ Log failed logins and sensitive actions
if (!validUser) {
  logger.warn(`Failed login for ${req.body.username}`);
}
logger.info(`User ${req.user.id} updated profile`);
```

---

## 10. **What is Server-Side Request Forgery (SSRF) and how do you prevent it?**

**Answer:** SSRF occurs when an attacker tricks the server into making requests to unintended destinations, including internal-only resources. Prevent it by whitelisting allowed destinations and rejecting requests that resolve to private/internal IP ranges.

```ts
// ✅ Whitelist allowed destinations
const allowed = ["https://api.example.com"];
if (!allowed.includes(req.query.url)) return res.status(400).send("Blocked");
axios.get(req.query.url);

// ✅ Reject requests resolving to internal/private IP ranges
const hostname = new URL(req.query.url).hostname;
const ip = (await dns.lookup(hostname)).address;
if (ip.startsWith("169.254") || ip.startsWith("127.") || ip.startsWith("10.")) {
  throw new Error("SSRF blocked");
}
```

---

## Summary

- Apply the **principle of least privilege**
- Use **automated scanning tools** (dependency and static analysis)
- Perform **secure code reviews**
- Build with **security by design**, not as an afterthought
