# Application Security

`govulncheck` (see `references/tooling.md`) and image scanning (see `references/docker.md`) handle
*known-CVE* exposure. This file covers the application-layer surface those tools don't see:
transport, authentication, authorization, input validation, and secret handling.

## TLS

**Where TLS terminates determines what your service must do.**

- **Terminated at the ingress** (k8s Ingress, ALB, Envoy, service mesh): your service speaks plain
  HTTP/gRPC inside the cluster. Trust the `X-Forwarded-*` headers *only* if the ingress strips
  client-supplied versions. Verify with the ingress operator — this is a frequent spoofing vector.
- **Terminated in your service** (edge service, no ingress, mTLS): you own the TLS config.
- **mTLS for service-to-service** inside a mesh: typically managed by the mesh (Istio, Linkerd) and
  your code stays plaintext-looking. If hand-rolled, use `crypto/tls` with `ClientAuth:
  tls.RequireAndVerifyClientCert` plus a strict `ClientCAs` pool.

When you do own the TLS config, keep it boring:

```go
tlsCfg := &tls.Config{
	MinVersion:       tls.VersionTLS12,                  // TLS 1.3 preferred; 1.2 floor for compat
	CurvePreferences: []tls.CurveID{tls.X25519, tls.CurveP256},
	CipherSuites:     nil,                               // let stdlib pick; only override if compliance forces
}
```

Don't disable cert verification in clients (`InsecureSkipVerify: true`) outside of explicit test
code. `gosec` flags this.

## Authentication

### OIDC / JWT (most common)

The pattern: an IdP (Okta, Auth0, Keycloak, Cognito, your own) issues JWTs signed by keys published
at a JWKS URL. Your service verifies signature, issuer, audience, expiry on every request.

```go
import (
	"github.com/coreos/go-oidc/v3/oidc"
)

// At startup — once. The provider caches JWKS and refreshes them automatically.
provider, err := oidc.NewProvider(ctx, cfg.Auth.IssuerURL)
verifier := provider.Verifier(&oidc.Config{ClientID: cfg.Auth.Audience})

// In auth middleware:
func AuthMiddleware(v *oidc.IDTokenVerifier) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			raw, ok := bearerToken(r) // "Authorization: Bearer ..." → token, true
			if !ok {
				http.Error(w, "missing bearer token", http.StatusUnauthorized)
				return
			}
			token, err := v.Verify(r.Context(), raw)
			if err != nil {
				http.Error(w, "invalid token", http.StatusUnauthorized)
				return
			}
			var claims struct {
				Sub    string   `json:"sub"`
				Email  string   `json:"email"`
				Scopes []string `json:"scope"`
			}
			_ = token.Claims(&claims)
			ctx := context.WithValue(r.Context(), principalKey, claims)
			next.ServeHTTP(w, r.WithContext(ctx))
		})
	}
}
```

- **Always verify**: signature, `iss`, `aud`, `exp`, `nbf` if present. `go-oidc` does this for you.
- **Cache JWKS** — `go-oidc`'s `NewProvider` does. Don't hand-fetch JWKS on every request.
- **Rotate keys** by relying on `kid` lookup against the cached JWKS, not by pinning a key in
  config.
- **Don't accept tokens via query string** — they end up in access logs and proxy histories.
- **Short token lifetimes** (5–15 min) + refresh tokens beats long-lived bearer tokens in blast
  radius if leaked.

### mTLS

For pure service-to-service inside an infrastructure you control, mTLS is simpler than JWT (no
issuer, no JWKS, no clock skew handling). Identity is the client cert's Subject/SAN; pair it with
an `allowlist` of acceptable identities per route.

### API keys

Acceptable for machine-to-machine where mTLS isn't viable. Treat them as secrets:
random ≥256 bits, hashed at rest (Argon2id or bcrypt), one rotation procedure that overlaps
old+new keys for a window.

## Authorization (separate from authentication)

Authentication answers *who is this?*; authorization answers *what may they do?*. Keep them in
separate middleware (or separate layers). **Enforce authorization in the service layer**, not the
handler — same business rule must apply whether the call came in over HTTP, gRPC, or a worker job.

Avoid sprinkling role checks across handlers; centralize in a `Policy` (or `Authz`) interface that
takes `(principal, action, resource)` and returns a decision. For non-trivial cases use OPA
(`open-policy-agent/opa`) or Cedar — both have Go SDKs.

## Input validation

Two layers, both required:

1. **Boundary parsing** rejects malformed input *cheaply* — close the connection / return 400
   without doing any business logic.
2. **Domain validation** enforces business rules (amount > 0, email matches a real format for your
   product, status transitions are legal).

### Boundary

- `http.MaxBytesReader` on every body. Without it a single curl can OOM your process.
- `json.Decoder` + `DisallowUnknownFields()` for strict APIs — catches typos before they become
  silent bugs.
- Reject oversized headers via `http.Server.MaxHeaderBytes`.
- Length-limit every free-text string field before storage (DB column lengths are not enough — they
  fail late and verbose).

### Domain

`go-playground/validator/v10` for declarative struct validation:

```go
type CreateOrderReq struct {
	Email   string `json:"email"   validate:"required,email,max=254"`
	Amount  int64  `json:"amount"  validate:"required,gt=0,lte=10000000"` // cents
	Country string `json:"country" validate:"required,iso3166_1_alpha2"`
}
```

Translate validator errors into your `ValidationError` type (`references/errors-and-concurrency.md`)
so the HTTP/gRPC mapping is uniform.

**Never trust input from the URL, headers, body, or context** to be safe for downstream use without
re-encoding for the target:

- SQL → parameterized queries (pgx does this; never concatenate).
- Shell exec → don't, but if you must: `exec.Command(name, args...)` with arg-array, never `sh -c`.
- HTML → `html/template`, never `text/template`, for HTML output.
- Log lines → strip CR/LF so an attacker can't inject fake log entries.

## CORS

Only relevant for browser-facing APIs. Don't echo `Access-Control-Allow-Origin: *` *and* allow
credentials — modern browsers reject the combination, and even where they don't, it's the
classic foot-gun.

Use `github.com/rs/cors` or the stdlib pattern manually:

```go
mux.Handle("/api/", corsMiddleware(cfg.CORS.AllowedOrigins, apiHandler))
```

Whitelist origins from config; never compute them from the `Origin` header at request time.

## CSRF

Only relevant for **cookie-based** session auth on state-changing endpoints. If you only accept
`Authorization: Bearer ...` tokens, CSRF doesn't apply (the browser doesn't auto-send `Authorization`
headers cross-origin).

If you do use cookies: `SameSite=Lax` (or `Strict`) cookies are the primary defense; add a
double-submit token (`gorilla/csrf`) for older browsers or strict compliance regimes.

## HTTP security headers

For browser-facing endpoints, set defaults in middleware:

```go
func SecurityHeaders(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		h := w.Header()
		h.Set("X-Content-Type-Options", "nosniff")
		h.Set("X-Frame-Options", "DENY")
		h.Set("Referrer-Policy", "no-referrer")
		h.Set("Strict-Transport-Security", "max-age=31536000; includeSubDomains")
		// CSP belongs here too, but it's app-specific — set per-route.
		next.ServeHTTP(w, r)
	})
}
```

Internal-only gRPC services don't need these (no browser).

## Secrets

(Extends `references/configuration.md`'s "Secrets" section — that file covers the *env-var
plumbing*; this one covers the *operational hygiene*.)

- **Mounted files > env vars** for k8s Secrets when the consumer can re-read on change. Env vars
  are captured at process start and frozen for the process lifetime; a mounted file rotates without
  redeploy.
- **Watch mounted secrets** with `fsnotify` if the consumer can hot-reload (DB DSN, JWKS, TLS
  certs). For tokens that the runtime caches (database connections, OAuth providers), schedule a
  re-init on rotation.
- **Rotation cadence**: short-lived workload identity (k8s ServiceAccount tokens, IAM IRSA) is
  the modern default — minutes-to-hours TTL, automatic refresh. Long-lived static credentials are
  technical debt with an expiration date.
- **No secrets in logs, ever.** Use the `Secret` type from `references/configuration.md`. Log
  redaction at the sink (Vector/Fluent Bit) is a backstop, not a primary defense.
- **No secrets in error messages returned to clients.** `internal error` is the only safe default;
  log the detail server-side and correlate via request ID.
- **No secrets in container env** visible via `docker inspect` or k8s `describe`. Use Secret
  references, not literal env values, in the Pod spec.

## Rate limiting & abuse

(Cross-link: `references/http-server.md` covers the resilience patterns; the security angle is
distinct.)

- Per-identity rate limits (authenticated user, API key, IP) are an authorization decision dressed
  up as a resilience pattern. Make them explicit.
- Login endpoints need stricter limits than read endpoints — credential stuffing is the most
  common automated attack and the cheapest one to mitigate.
- Lockout-after-N policies must be designed with **account enumeration** in mind: don't tell the
  attacker "this account is locked" — return the same 401 you'd return for a wrong password.

## Anti-patterns

- `InsecureSkipVerify: true` outside test code.
- JWT verification that skips `aud` or `exp`. Both have caused real production incidents.
- Bearer tokens accepted via query string or stored in browser localStorage (XSS-readable).
- Authorization checks in handlers but not in the corresponding worker / cron / Kafka consumer
  paths — same business action, different entry point, no policy enforcement.
- Logging request bodies on errors (eats tokens and PII).
- Embedding secrets in container images, in `ARG`s, or in Helm `values.yaml` checked into git.
- `text/template` for HTML output.
- "Soft" rate limits that increment a counter but never reject — they're metrics, not defense.
- Redacting only the obvious fields (`password`) and missing the new ones (`api_key`, `refresh_token`).
  Use a type that self-redacts (`Secret`) so adding a new secret field is impossible to miss.
