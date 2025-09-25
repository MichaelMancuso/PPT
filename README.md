# PPT
Premonotion Penetration Testing GitPages Site

0) Setup & Guardrails (quick)

Confirm scope (hosts, APIs, environments), auth types (cookie, JWT, OAuth/OIDC, SSO), roles to test, and rate-limit/SLA tolerances.

Prepare test identities (e.g., user, manager, admin, cross-tenant user) and a clean baseline browser profile behind Burp/ZAP.

Turn on full logging (Burp project file, screenshot tool, timestamped notes).

1) Enumeration: Map the Attack Surface
1.1 Crawl & Collect

Passive first: browse like a user through all flows (signup, login, forgot-password, 2FA enroll/disable, profile edit, payments, file uploads, admin screens).

Proxy capture (Burp): group traffic by host → path → method; save all requests.

Spider/Discover:

Parse JS for hidden endpoints, feature flags, API base URLs, GraphQL operation names.

Check /robots.txt, /sitemap.xml, /.well-known/ (security.txt, apple-app-site-association), /openapi.json, /swagger.json, /graphql (introspection), /actuator (Spring), /health.

Try verb variations (GET/POST/PUT/PATCH/DELETE/OPTIONS), alternative content types (JSON ↔ form-urlencoded ↔ multipart).

Subsurfaces:

API (REST/GraphQL/gRPC-gateway), WebSockets/SSE, background jobs (export, email), third-party callbacks (OAuth redirect URIs, payment webhooks), CDN edges, file storage (S3/GCS signed URLs).

1.2 Session & Identity Modeling

Record cookies (Secure, HttpOnly, SameSite, Path/Domain scope), JWT claims (iss, aud, kid, alg), CSRF tokens (where generated & validated).

Note tenant identifiers (org_id, account_id), resource IDs (user_id, order_id), and role gates (is_admin, is_premium).

1.3 Build a Test Map

Create a lightweight matrix:
[Method] [Path] [Auth?] [Role needed] [Params] [Stateful?] [Side effects] [Rate limit?] [Notes/risk]

This becomes your “to-test” backlog.

2) Parameter Discovery & Classification
2.1 Find Hidden Inputs

Watch for params added by the browser/framework: _method, _csrf, __RequestVerificationToken, __EVENTVALIDATION, g-recaptcha-response.

Use Burp Param Miner (or manual dictionary) against headers, cookies, query, body, JSON keys. Try things like:

X-Original-URL, X-Rewrite-URL, X-HTTP-Method-Override

X-Forwarded-Host/For/Proto, Forwarded

X-Api-Key, X-Auth-Token, X-User-Id

For APIs, compare OpenAPI/Swagger vs. observed traffic → discrepancies often hide endpoints.

2.2 Classify “Interesting” Params (and why)

Identifiers: user_id, account_id, order_id → test IDOR/BOLA.

Money/Quantity: price, qty, discount, credit → test price tampering & overflow.

Role/Flags: is_admin, approved, beta → test priv-escalation.

URL/Host: callback, redirect_uri, image_url, feed → test SSRF/Open Redirect.

File fields: filename, content_type → test upload RCE, traversal, polyglots.

Filters/Sort: sort, order, fields → test mass data exposure, SQL/NoSQL, GraphQL field over-fetch.

Search/Free text → test XSS, injection, ReDoS.

3) Fuzzing Strategy (targeted & disciplined)

Goal: trigger authZ errors, state errors, validation failures, and edge-case bugs—not random noise.

3.1 Structural Fuzz (per type)

Numeric

-1, 0, 1, 2, 2147483647, 2147483648, 9999999999

Non-int: 1.0, 1e9, NaN
Checks: off-by-one, overflow, sign, business rule bypass (min/max).

Booleans

Toggle true/false, 1/0, "true"/"false", remove the key, send null.

Strings

Empty, single char, long (2–8 KB), mixed unicode, %00, control chars, CRLF \r\n.

Templating markers: ${{7*7}}, {{7*7}}, <%= %>, {{constructor.constructor('return process')()}}.

Paths

../../etc/passwd, ..%2f..%2f, %2e%2e/, double encodings; probe path traversal.

Content-Type Flip

Send JSON with application/x-www-form-urlencoded and vice-versa; remove Content-Type.

Encoding/Case

Case variations, duplicate keys, mixed encodings (URL-encoded twice, UTF-7 for legacy).

Headers

X-Forwarded-For: 127.0.0.1, X-Forwarded-Host: admin.internal, Origin: https://evil.tld, X-HTTP-Method-Override: PUT.

File Uploads

Extension vs MIME mismatch; test.php.jpg, SVG with embedded script, PDF/ZIP polyglots, large files, zip bombs (careful!).

SSRF payload seeds

http://127.0.0.1:80/, http://169.254.169.254/latest/meta-data/, http://[::1], file://, gopher://, dict:// (check scheme filters).

3.2 Behavioral/State Fuzz

Out-of-order: call step 3 before step 2 (workflow bypass).

Replay: resend a successful POST (idempotence). Does it duplicate, re-credit, or re-send?

Concurrency: burst N identical requests (race conditions).

Role swap: repeat an admin action with a basic user’s token.

Cross-tenant: change org_id to another tenant’s ID.

3.3 Signals to Watch

Status codes: 401/403/404/409 vs 200; differences by role.

Timing: slower responses can hint at server-side fetching/SSRF.

Response diffs: extra fields, error stacks, leaked IDs/emails.

4) Systematic Function Testing (repeatable recipe)

For each function/endpoint/flow, run this 10-step loop:

Happy Path
Verify the intended behavior with valid input (establish your oracle).

AuthN/Session

Use no auth, expired auth, other user’s auth, and different roles.

Check cookie flags (Secure/HttpOnly/SameSite), JWT alg/kid, logout token invalidation.

AuthZ

Change every resource identifier (user_id, order_id), both in body and in URL.

Try “list” endpoints returning other tenants’ data (BFLA/BOLA).

Input Validation

Apply targeted payloads from §3 per parameter type.

Try wrong content types and missing required fields.

State/Workflow

Skip required steps; repeat steps; alter sequence; reuse CSRF tokens across flows.

Rate Limiting

Rapid-fire requests (increase RPS), observe 429s or lack thereof; try header rotation (X-Forwarded-For) to probe per-IP rules.

Idempotence & Replay

Re-POST purchases, refunds, password resets; check duplicate effects.

Business Rules

Boundary tests (price < min, qty > max), free-trial abuse, coupon stacking, daily limits.

Side Channels

Webhooks: can you point them to your server? Are signatures verified?

CORS: send Origin: https://evil.tld and check Access-Control-Allow-*.

Cache: test cache-poison vectors (vary headers, reflected input in responses).

Evidence & Fix Hints

Save the minimal PoC, timestamp, request/response, and 1-2 line root cause & mitigation (e.g., “Missing server-side state check—enforce payment_complete before confirm_order”).

5) High-Value Flows (never skip)

Signup / Invite / Onboarding (multi-use emails? reuse invite tokens?).

Login / MFA / Session (brute, bypass, disable flows).

Password Reset / Email Change (token leakage, token replay, open redirect, CSRF).

Role/Permission Admin (RBAC gaps, UI-only checks).

Payment/Wallet/Orders (price tamper, coupon abuse, refund fraud, split transactions).

File Upload / Import/Export (RCE, path traversal, data leakage).

Search/Reporting (mass data exposure, over-fetching).

6) Lightweight Tools/Commands (manual-first, automation-assisted)

ffuf (dirs & APIs)
ffuf -u https://app.tld/FUZZ -w wordlist.txt -recursion -mc all -fc 404
ffuf -u https://api.tld/v1/users?search=FUZZ -w payloads.txt -mc all

curl method/ctype flips
curl -X PUT https://app.tld/resource -d 'x=1' -H 'Content-Type: application/json'
curl -X POST https://app.tld/path -d @body.json -H 'Content-Type: application/x-www-form-urlencoded'

Concurrency probe (bash)
for i in {1..20}; do curl -s -X POST ... & done; wait

GraphQL
Try introspection; send queries selecting extra fields or nested connections to test over-exposure/DoS.

7) Interview-friendly “Why this matters” phrasing

“I first build a map of the application—endpoints, roles, states—so my testing is deliberate, not random.”

“I classify parameters (IDs, prices, URLs) because each family has distinct abuse patterns.”

“For every function I run a fixed loop: authN → authZ → validation → state → rate limit → idempotence → business rules.”

“My signal is differential behavior: status codes, response shapes, and timing—then I minimize to a clean PoC and remediation.”

8) Compact Checklist (copy/paste)
