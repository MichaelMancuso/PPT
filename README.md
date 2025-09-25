# Web Application Penetration Testing: Manual Testing Playbook

A comprehensive guide for mapping attack surface, fuzzing parameters, and testing each function systematically. Perfect for interview prep **and** real-world engagements.

---

## 0️⃣ Setup & Guardrails (Quick)
- **Confirm Scope:** Hosts, APIs, environments, roles, rate-limit/SLA tolerances.
- **Prepare Test Identities:** Accounts for each role (user, admin, cross-tenant user).
- **Proxy Setup:** Burp Suite/ZAP capturing all traffic.
- **Logging:** Save Burp project file, take screenshots, timestamped notes.

---

## 1️⃣ Enumeration: Mapping the Attack Surface

### 1.1 Crawl & Collect
- **Passive First:** Browse like a user through all flows: signup, login, forgot-password, 2FA, profile edit, payments, file uploads, admin screens.
- **Proxy Capture:** Group traffic by host → path → method; save all requests.
- **Spider/Discover:**
  - Parse JavaScript for hidden endpoints, feature flags, API base URLs.
  - Check `/robots.txt`, `/sitemap.xml`, `/.well-known/` (security.txt, apple-app-site-association), `/openapi.json`, `/swagger.json`, `/graphql`, `/actuator`, `/health`.
  - Try verb variations (GET/POST/PUT/PATCH/DELETE/OPTIONS), alternative content types.
- **Subsurfaces:** APIs, WebSockets, background jobs, third-party callbacks (OAuth redirect URIs, payment webhooks), CDN edges, file storage (S3/GCS).

### 1.2 Session & Identity Modeling
- Record cookies: Secure, HttpOnly, SameSite, Path/Domain scope.
- Note JWT claims (iss, aud, kid, alg), CSRF token generation/validation.
- Identify tenant identifiers, resource IDs, and role-gates.

### 1.3 Build a Test Map
Create a lightweight matrix of endpoints with:
- Method, Path, Auth Required, Role, Params, Stateful?, Side Effects, Rate Limit?, Notes

---

## 2️⃣ Parameter Discovery & Classification

### 2.1 Find Hidden Inputs
- Use Burp Param Miner or manual fuzzing to discover headers, cookies, query/body params.
- Candidate headers: `X-Original-URL`, `X-Rewrite-URL`, `X-Forwarded-Host`, `X-HTTP-Method-Override`, `X-Api-Key`, `X-Auth-Token`.
- Compare API schemas vs observed traffic.

### 2.2 Classify Parameters
- Identifiers → IDOR/BOLA
- Money/Quantity → Price tampering, overflow
- Role/Flags → Privilege escalation
- URL/Host → SSRF, open redirect
- File fields → Upload RCE, traversal
- Search/Filter → Data exposure, SQLi, GraphQL over-fetch

---

## 3️⃣ Parameter Fuzzing & Payload Strategy

### Numeric Payloads
```
-1, 0, 1, 2, 2147483647, 2147483648, 9999999999, 1.0, 1e9, NaN
```

### Boolean Payloads
```
true, false, 1, 0, "true", "false", null, missing
```

### String Payloads
```
<script>alert(1)</script>, ../../etc/passwd, %00, CRLF, long unicode, {{7*7}}, ${7*7}
```

### Content-Type Flips
- Send JSON with `application/x-www-form-urlencoded` header and vice-versa.
- Remove Content-Type entirely.

### Header Payloads
```
X-Forwarded-For: 127.0.0.1
X-Forwarded-Host: admin.internal
Origin: https://evil.tld
```

### SSRF Payloads
```
http://127.0.0.1, http://169.254.169.254/latest/meta-data/, file:///, gopher://
```

### File Upload Tests
- Extension vs MIME mismatch
- Polyglot payloads (PHP+JPEG)
- SVG with embedded scripts
- Large files / zip bombs (safe env)

### Workflow & State Fuzzing
- Skip steps, replay requests, reorder, concurrent requests.
- Switch roles/tokens mid-flow.

---

## 4️⃣ Systematic Function Testing Process

**For each function or endpoint:**

1. **Happy Path** – Verify intended behavior.
2. **AuthN/Session** – No auth, expired auth, other user’s token.
3. **AuthZ** – Modify IDs, test cross-tenant access.
4. **Input Validation** – Flip types, send injection payloads.
5. **State/Workflow** – Skip steps, replay, out-of-order.
6. **Rate Limiting** – Burst requests, look for 429.
7. **Idempotence** – Replay transactions, check duplication.
8. **Business Rules** – Min/max abuse, free trial loops, coupon stacking.
9. **Side Channels** – CORS misconfig, cache poisoning, open redirects.
10. **Evidence & Fix** – Minimal PoC, root cause, remediation hint.

---

## 5️⃣ High-Value Flows to Test
- Signup, invite, onboarding tokens
- Login, MFA, session management
- Password reset & email change
- Role/permission management
- Payments, coupons, refunds
- File upload/import/export
- Search/reporting with bulk data

---

## 6️⃣ Tools & Quick Commands

### ffuf
```bash
ffuf -u https://target/FUZZ -w wordlist.txt -recursion -mc all -fc 404
```

### curl Method/Type Flip
```bash
curl -X PUT https://app.tld/resource -d 'x=1' -H 'Content-Type: application/json'
curl -X POST https://app.tld/path -d @body.json -H 'Content-Type: application/x-www-form-urlencoded'
```

### Concurrency Probe
```bash
for i in {1..20}; do curl -s -X POST ... & done; wait
```

### GraphQL Testing
- Run introspection
- Over-fetch sensitive fields: `{ users { passwordHash, apiKey } }`

---

## 7️⃣ Interview-Ready Talking Points
- *“I build an attack surface map of hosts, endpoints, parameters, roles, and states.”*
- *“I classify parameters so I can target IDOR, SSRF, price tampering, etc.”*
- *“I use a repeatable 10-step loop per endpoint covering authN, authZ, validation, state, rate limits, and business logic.”*
- *“I look for differential behavior: status codes, response diffs, timing. Then I write a minimal PoC and suggest a fix.”*

---

## 8️⃣ Compact Checklist
- [ ] Crawl & capture all traffic
- [ ] Extract & classify parameters
- [ ] Model sessions, roles, tenants
- [ ] Test authN/authZ on all endpoints
- [ ] Apply payloads per parameter type
- [ ] Skip/replay/out-of-order tests
- [ ] Check rate limits & concurrency
- [ ] Test CORS/CSRF/cache/redirects
- [ ] Abuse file uploads (safe env)
- [ ] Document PoC + mitigation
