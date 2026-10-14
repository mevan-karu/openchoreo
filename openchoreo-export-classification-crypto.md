# OpenChoreo: Encryption Classification Answers (b)(1)–(8)

Verified against `main` at commit **`e76c145ca`**, Go toolchain **1.26.3**, CPython **3.14**, and the k3d single-cluster install (`install/k3d/single-cluster/`).

File references are relative to the repository root, except those prefixed `$GOROOT`, which point at the Go 1.26.3 standard library source.

**Scope note.** OpenChoreo is built in three languages, and they don't share a cryptographic stack:

| Part | Language | Crypto provider |
|---|---|---|
| 9 server binaries and the CLI (`cmd/`) | Go 1.26.3 | Go standard library, statically linked |
| 3 agent services (`agents/`): sre-agent, finops-agent, portal-assistant | CPython 3.14 | **OpenSSL** via Python `ssl`, plus `cryptography` 50.0.0 |
| Console UI (`ghcr.io/openchoreo/openchoreo-ui`) | Node.js, built in a separate repository | Node/OpenSSL — not analyzed here |

---

## Summary: corrections to the draft answers

The draft answers appear to have been carried over from another product. Checked against the code, several are wrong:

| Draft claim | Finding in OpenChoreo | Evidence |
|---|---|---|
| AES-256 used, "encryption mode is cipher block chaining" | OpenChoreo's own code never calls AES, and imports neither `crypto/aes` nor `crypto/cipher`. AES appears only inside TLS. The preferred modes are **AEAD (AES-GCM, ChaCha20-Poly1305)**; AES-CBC is only a lower-preference TLS 1.2 fallback. | §1.1; `$GOROOT/src/crypto/tls/cipher_suites.go:283-312`, `defaults.go:61-96` |
| RSA 4096 | **2048 and 4096** are both in use, and several certificates take cert-manager's default of RSA 2048 | §1.3 |
| DHE 2048 | **Not used.** Go's `crypto/tls` has no finite-field DHE. Key exchange is ECDHE plus **hybrid post-quantum ML-KEM**. | `$GOROOT/src/crypto/tls/defaults.go:21-35` |
| SHA-256 only | Also **Ed25519 (EdDSA)**, **ECDSA P-256/384/521**, **RSA-PSS**, **HMAC-SHA256**, **HMAC-SHA1** (TLS CBC fallback), and a CSPRNG | §1.2 |
| "Encryption is handled by the standard crypto libraries offered by Go" | True for the Go binaries only. The three Python agents use **OpenSSL** and `cryptography` 50.0.0. | §1.5, §8 |
| "No API or capabilities for the user to modify" | Operators can disable TLS verification, supply their own certificates and keys, and — in the agents — **disable JWT signature verification entirely** | §7 |
| Protocols "TLS, SSL" | **SSL is not supported.** TLS 1.2 is the effective floor and TLS 1.3 is supported. One internal path (gRPC to Hubble) is **plaintext by design**. | §6 |

---

## (1) Symmetric and asymmetric algorithms, key lengths, usage, and modes

### 1.1 Go crypto packages used (import census)

Counts of files importing each package, excluding `.worktree/` and `.claude/worktrees/`. Format: all files [non-test files].

| Package | Files | Non-test locations |
|---|---|---|
| `crypto/tls` | 40 [16] | TLS/mTLS clients and servers across gateway, agents, API, observer |
| `crypto/x509` | 22 [12] | Certificate parsing, CA pools, client-certificate authentication |
| `crypto/x509/pkix` | 9 [1] | `remoteconnect_provision.go` (certificate subject) |
| `crypto/rand` | 14 [6] | Key generation, PKCE, serials, IDs |
| `crypto/ed25519` | 13 [4] | Capability token signing and verification |
| `crypto/sha256` | 11 [8] | HMAC, PKCE S256, fingerprints, deterministic names |
| `crypto/ecdsa`, `crypto/elliptic` | 6 [1] each | `remoteconnect_provision.go` (P-256 key generation) |
| `crypto/rsa` | 3 [1] | `jwt/jwks.go` (verifying OIDC token signatures) |
| `crypto/hmac` | 3 [1] | `services/git/signature.go` (webhook signatures) |

**Not imported anywhere:** `crypto/aes`, `crypto/cipher`, `crypto/des`, `crypto/md5`, `crypto/sha1`, `crypto/sha512`, `crypto/ecdh`, `crypto/mlkem`, and `golang.org/x/crypto`. (`golang.org/x/crypto` v0.53.0 is in `go.mod` as `// indirect` only, reached through `go-playground/validator` → `sha3`.)

**Non-cryptographic randomness**, stated for completeness: `math/rand` in `internal/controller/renderedrelease/controller.go:682` and `math/rand/v2` in `internal/cluster-gateway/fabric/mesh.go:524` and `internal/cluster-agent/agent.go:227`, all for retry jitter and backoff, none security-relevant.

**The Go code performs no application-level encryption.** There is no call that encrypts a payload or data at rest. All confidentiality comes from TLS.

### 1.2 Algorithm table

| Algorithm | Mode / variant | Key size (bits) | Purpose | Where |
|---|---|---|---|---|
| **AES** | GCM (TLS 1.2 and 1.3); CBC with HMAC-SHA1 (TLS 1.2 fallback only) | 128, 256 | Confidentiality and integrity in transit | Negotiated inside Go `crypto/tls`; never called directly by OpenChoreo |
| **ChaCha20-Poly1305** | AEAD | 256 | Confidentiality and integrity in transit | Go default suite, preferred without AES hardware (`cipher_suites.go:314-316`) |
| **ECDHE** | X25519, P-256, P-384, P-521 | 255, 256, 384, 521 | TLS key agreement | `$GOROOT/src/crypto/tls/defaults.go:31-32` |
| **ML-KEM (hybrid)** | X25519MLKEM768, SecP256r1MLKEM768, SecP384r1MLKEM1024 | ML-KEM-768 / -1024 combined with an ECDH group | Post-quantum hybrid TLS 1.3 key agreement, first preference by default | `$GOROOT/src/crypto/tls/defaults.go:31` |
| **RSA** | PKCS#1 v1.5 and PSS signatures | 4096 (CAs), 2048 (leaf, client, and defaulted certificates) | Certificate signatures, TLS authentication, verifying OIDC tokens | §1.3; `internal/server/middleware/auth/jwt/jwks.go:143-170` |
| **RSA (JWT, Go services)** | RS256/384/512 and PS256/384/512 both accepted | Set by the identity provider | Authenticating API and Observer requests | §7; non-RSA JWKS keys are skipped (`jwks.go:101`) |
| **RSA / ECDSA (JWT, Python agents)** | RS256/384/512, ES256/384/512 | Set by the identity provider | Authenticating agent requests | `agents/common/src/common/auth/jwt.py:79` |
| **ECDSA** | P-256 with SHA-256 | 256 | Self-signed TLS server certificate per `occ remote` agent | `internal/openchoreo-api/api/handlers/remoteconnect_provision.go:592-627` |
| **Ed25519 (EdDSA)** | Pure Ed25519 | 256 | Signing and verifying remote-connect capability JWTs; also a TLS signature scheme | `internal/remoteconnect/capability.go:16,151-191`; key loaded as PKCS#8 at `remoteconnect.go:1100-1113` |
| **HMAC-SHA256** | n/a | Length of the operator-supplied webhook secret | Authenticating inbound Git webhooks, compared in constant time | `internal/openchoreo-api/services/git/signature.go:17-37` |
| **HMAC-SHA1** | n/a | 160 | Record MAC, used only if a TLS 1.2 `..._CBC_SHA` fallback suite is negotiated | `$GOROOT/src/crypto/tls/cipher_suites.go:290-291` |
| **SHA-256** | n/a | 256 | PKCE S256 challenge; certificate fingerprint; CA grouping; deterministic non-secret names and cache keys | `internal/occ/auth/pkce.go:67`; `remoteconnect_provision.go:629`; `internal/cluster-gateway/server.go:1491`; `internal/dataplane/kubernetes/name.go:68`; `internal/occ/cmd/remote/fetch.go:90`; `internal/controller/releasebinding/delivery_events.go:724`; `pkg/fsindex/cache/cache.go:220,227`; Python `hashlib.sha256` at `agents/portal-assistant/src/clients/mcp.py:199,220` and `src/api/agent_routes.py:39` |
| **SHA-384 / SHA-512** | n/a | 384 / 512 | TLS signature schemes and TLS 1.3 HKDF key derivation | `$GOROOT/src/crypto/tls/defaults.go:41-55` |
| **CSPRNG** | Go `crypto/rand`; OS entropy | n/a | Key generation, 128-bit certificate serials, 256-bit PKCE verifier, 128-bit state, IDs and name suffixes | `remoteconnect_provision.go:593,597`; `internal/occ/auth/pkce.go:58-75`; `internal/cluster-gateway/server.go:150-157`; `internal/openchoreo-api/services/workflowrun/service.go:1020`; `internal/pipeline/workflow/pipeline.go:393` |

**TLS signature schemes** advertised by default also include Ed25519 and RSA-PSS (`defaults.go:41-55`). SHA-1 signature schemes appear in that list but are **rejected at verification** unless `GODEBUG=tlssha1=1` (`$GOROOT/src/crypto/tls/auth.go:244-256`; the godebug is declared at `common.go:1762`). That is separate from HMAC-SHA1, which remains live as the record MAC in the CBC fallback suites.

### 1.3 Certificate key sizes (issued by cert-manager)

| Certificate | Algorithm / size | Role | File |
|---|---|---|---|
| Cluster-gateway CA | RSA 4096, `rotationPolicy: Always` | Root CA for agent ↔ gateway TLS | `openchoreo-control-plane/templates/cluster-gateway/ca-certificate.yaml:21-24` |
| Cluster-gateway internal CA | RSA 4096, `rotationPolicy: Never`, **subordinate** to the gateway CA | Signs internal-API mTLS client certificates | `cluster-gateway/internal-ca.yaml:23-33` |
| Observability-plane CA | RSA 4096, 10-year duration | Observability TLS CA (when `tls.enabled`) | `openchoreo-observability-plane/templates/tls/certificate.yaml:12-22` |
| Observability-plane wildcard | **Unset → cert-manager default: RSA 2048** | Serving certificate | `openchoreo-observability-plane/templates/tls/certificate.yaml:29-41` |
| openchoreo-api → gateway client | RSA 2048 | mTLS client | `openchoreo-api/cluster-gateway-client-certificate.yaml:19-21` |
| controller-manager → gateway client | RSA 2048 | mTLS client | `controller-manager/cluster-gateway-client-certificate.yaml:18-20` |
| Cluster-agent certificate (data, workflow, observability planes) | RSA 2048, `client auth` | mTLS client identity per plane | `openchoreo-data-plane/templates/cluster-agent/certificate.yaml:17-19` and the equivalents in the other two charts |
| Cluster-gateway serving certificate | **Unset → RSA 2048** | TLS server | `cluster-gateway/certificate.yaml` (no `privateKey` block) |
| Controller-manager webhook | **Unset → RSA 2048** | Admission webhook TLS | `controller-manager/webhook-server-cert.yaml` |
| Data-plane webhook serving certificate | **Unset → RSA 2048** | Webhook TLS | `openchoreo-data-plane/templates/security/serving-cert.yaml` |
| Remote-agent certificate (`occ remote`) | ECDSA P-256, generated in Go | TLS server | `remoteconnect_provision.go:593` |
| Remote-connect capability signing key | Ed25519, generated by Helm | JWT signing | `openchoreo-api/remote-connect-signing-key.yaml:11` |

cert-manager defaults, where `privateKey` is unset: **RSA**, size **2048**, **PKCS#1** encoding, and `rotationPolicy: Always` since cert-manager 1.18 (GA in 1.19). The single-cluster guide pins **v1.19.4**, so every certificate above that omits `privateKey` **generates a fresh private key on each renewal**. Sources: cert-manager website, `content/docs/reference/api-docs.md` (`CertificatePrivateKey`) and `content/docs/releases/release-notes/release-notes-1.18.md`.

### 1.4 Encryption modes supported

Nothing in the Go code sets `CipherSuites`, `CurvePreferences` or `MaxVersion`, and no chart sets `GODEBUG`, so the Go 1.26.3 defaults apply:

- **TLS 1.3:** `TLS_AES_128_GCM_SHA256`, `TLS_AES_256_GCM_SHA384`, `TLS_CHACHA20_POLY1305_SHA256` (`defaults.go:92-96`).
- **TLS 1.2**, in preference order after default filtering:
  1. `ECDHE_{ECDSA,RSA}_WITH_AES_{128,256}_GCM_SHA{256,384}` and `ECDHE_{ECDSA,RSA}_WITH_CHACHA20_POLY1305_SHA256`
  2. `ECDHE_{ECDSA,RSA}_WITH_AES_{128,256}_CBC_SHA`
- **Excluded by default:** RSA key-exchange and 3DES suites (unless re-enabled by `GODEBUG`), and RC4 and `*_CBC_SHA256` suites always (`defaults.go:69-76`; `cipher_suites.go:338-367`).

**Answer to "which modes":** GCM and ChaCha20-Poly1305 (both AEAD) are primary; CBC exists only as a TLS 1.2 fallback. CFB and CTR are not used.

### 1.5 Python agent cryptography

The three agent images (`agents/`) are CPython 3.14 (`.python-version`) installed by `uv` as a managed build (`UV_PYTHON_PREFERENCE=only-managed`), on `gcr.io/distroless/cc`:

| Element | Detail |
|---|---|
| TLS | Python `ssl` (OpenSSL) via `httpx` 0.28.1 for all outbound HTTP |
| JWT verification | `pyjwt[crypto]` 2.13.0 with `cryptography` 50.0.0, accepting **RS256/384/512 and ES256/384/512** (`agents/common/src/common/auth/jwt.py:79`) |
| JWKS fetching | `PyJWKClient` with a cached key set; verification can be turned off by building a context with `check_hostname = False` and `verify_mode = ssl.CERT_NONE` (`jwt.py:43-45`) |
| OAuth 2.0 | `authlib` 1.7.2, `client_credentials` grant with `client_secret_post` (`agents/common/src/common/auth/oauth_client.py:53,64,89,115,137,142`) |
| Hashing | `hashlib.sha256`, truncated to 16 hex characters for cache keys (`portal-assistant/src/clients/mcp.py:199,220`; `src/api/agent_routes.py:39`) |
| Server TLS | None. All three start `uvicorn` with no `--ssl-keyfile`, so they serve plain HTTP and TLS terminates at the gateway. |

`cryptography` 50.0.0 is installed from prebuilt manylinux/musllinux wheels (`agents/sre-agent/uv.lock`). **Whether OpenSSL is statically or dynamically linked in these images is not determined from this repository** — see open item 1.

---

## (2) How keys are generated and managed

OpenChoreo has **no custom key-management system**. Keys come from four sources.

1. **cert-manager** (v1.19.4, `install/k3d/single-cluster/README.md:38-42`) generates the RSA keys and X.509 certificates in §1.3.
   - **Issuer chain:** a `selfSigned` Issuer (`cluster-gateway/selfsigned-issuer.yaml:13`) issues the RSA 4096 gateway CA; a `ca` Issuer backed by the `cluster-gateway-ca` Secret (`cluster-gateway/issuer.yaml:11-12`) signs the leaf certificates. The **internal CA is subordinate, not a root** — it is issued by `<gateway>-ca-issuer` (`internal-ca.yaml:32-33`) and then backs its own Issuer for internal client certificates.
   - **Storage:** Kubernetes `Secret`s (`tls.crt`, `tls.key`, `ca.crt`), mounted into pods.
   - **Rotation:** cert-manager renews on schedule, with configurable `duration` and `renewBefore` (`cluster-gateway/certificate.yaml:30-35`). The gateway CA sets `rotationPolicy: Always`, so its key changes on renewal. The internal CA sets `rotationPolicy: Never`, and the template documents why: the gateway trusts only the current CA bundle and there is no staged trust rollout, so rotating that key would orphan already-issued client certificates until they are reissued (`internal-ca.yaml:26-31`). Certificates that omit `privateKey` inherit the 1.18+ default of `Always`.
   - **Trust distribution:** the public CA certificate is copied into a `cluster-gateway-ca` ConfigMap per plane namespace (`README.md:140-180`), and each plane registers its agent CA in its `Cluster*Plane` custom resource (`clusterAgent.clientCA`, `README.md:216-245`).
2. **Helm (Sprig `genPrivateKey "ed25519"`)** generates the Ed25519 capability-signing key at install time, stored as PKCS#8 PEM in the `remote-connect-signing-key` Secret (`openchoreo-api/remote-connect-signing-key.yaml:2-11`) and mounted at `/etc/remote-connect/signing.pem` (`values.yaml:1969,1975,1988`). This key is produced by Helm's template engine, **not** by Go code. Operators may supply their own (`generateSigningKey: false`), and a `kid` header supports rotation (`capability.go:151-160`).
3. **Go code** generates one key type at runtime: an **ECDSA P-256** key plus a self-signed X.509 certificate per remote agent, using `crypto/rand`, with a 128-bit random serial, `DigitalSignature | KeyEncipherment` usage, `ServerAuth` extended usage, PKCS#8 PEM encoding (`remoteconnect_provision.go:592-627`), reissued as it nears expiry or its SNI changes (`remoteconnect_provision.go:570-590`).
4. **The external identity provider** — **ThunderID chart 1.0.1** in the single-cluster guide (`README.md:86-90`) — owns the keys that sign access tokens. OpenChoreo only fetches public keys from JWKS (`jwks.go:72-123`; Python `PyJWKClient`), so the modulus size is the IdP's choice.

**Sizes and curves in use:** RSA 2048 and 4096; ECDSA P-256; Ed25519. TLS may additionally negotiate P-384, P-521, X25519, ML-KEM-768 and ML-KEM-1024.

---

## (3) Non-standard cryptography

**None.** Every primitive comes from a recognized standard, implemented by the Go standard library, OpenSSL, or `cryptography`:

| Algorithm | Standard |
|---|---|
| AES | FIPS 197 |
| GCM | NIST SP 800-38D |
| ChaCha20-Poly1305 | RFC 8439 |
| SHA-1 / SHA-2 | FIPS 180-4 |
| HMAC | FIPS 198-1 / RFC 2104 |
| RSA (PKCS#1 v1.5, PSS) | RFC 8017 |
| ECDSA | FIPS 186-5 |
| Ed25519 | RFC 8032 / FIPS 186-5 |
| X25519 | RFC 7748 |
| ML-KEM | FIPS 203 |
| TLS 1.2 / 1.3 | RFC 5246 / RFC 8446 |
| X.509 | RFC 5280 |
| JWT / JWS / JWK | RFC 7519 / 7515 / 7517 |
| OAuth 2.0, PKCE | RFC 6749 / RFC 7636 |

The only crypto-adjacent original code is how standard algorithms are *applied*: HMAC-SHA256 webhook verification following the GitHub `sha256=` convention (`services/git/signature.go`), and EdDSA-signed capability tokens (`remoteconnect/capability.go`). Neither introduces a new algorithm.

---

## (4) Pre-processing of plaintext before encryption

**None.** Nothing is compressed or interleaved before entering TLS, and Go's `crypto/tls` offers no TLS-level compression.

Encoding steps that are **not** encryption pre-processing: base64url (JWT segments, PKCE verifier and challenge, `internal/occ/auth/pkce.go:64-75`), PEM/DER (keys and certificates), and hex (signatures and hashes).

---

## (5) Post-processing of ciphertext after encryption

**None beyond the standard protocols.** TLS record framing happens inside the TLS library. WebSocket framing (`gorilla/websocket`), HTTP/1.1 and HTTP/2 framing all operate on plaintext inside the TLS channel, not on ciphertext. JWS compact serialization is signing, not encryption.

---

## (6) Communication and cryptographic protocols

| Protocol | Use | Evidence |
|---|---|---|
| **TCP** | Transport for everything | n/a |
| **HTTP/1.1, HTTP/2** | REST API, Observer API, agents, console, JWKS. HTTP/2 is disabled for the controller webhook and metrics servers unless `--enable-http2` is passed. | `cmd/main.go:383-390` |
| **TLS 1.2 / 1.3** (never SSL) | All TLS endpoints. Most configurations set `MinVersion: tls.VersionTLS12` explicitly; four rely on Go's default floor (see §7). | `cluster-gateway/server.go:190-193`; `fabric/mesh.go:338-344`; `cluster-agent/agent.go:276-284`; `remote-agent/server.go:354-356`; `remote-agent/authorize.go:34`; `remoteconnect/client.go:73`; `clients/gateway/client.go:214-215`; `clients/kubernetes/proxy_client.go:789-790`; `observer/service/uid_resolver.go:55-56` |
| **Mutual TLS (X.509 client certificates)** | Cluster agents → gateway, verified in the application against each plane's registered CA; gateway internal API on port 8444 with `RequireAndVerifyClientCert` (enabled by default); gateway mesh peers | `cluster-gateway/server.go:187-193,427`; `fabric/mesh.go:356`; `values.yaml:4061` |
| **WebSocket over TLS (`wss`)** | Persistent agent → gateway tunnel, `wss://cluster-gateway.openchoreo-control-plane.svc.cluster.local:8443/ws` | `openchoreo-data-plane/values.yaml:421`; `cluster-agent/agent.go:276` |
| **TLS with SNI routing** | `occ remote`: the router forwards TLS without terminating it; the remote agent terminates with its ECDSA certificate | `internal/remote-agent-router/router.go:4`; `remote-agent/server.go:200` |
| **gRPC — plaintext, no TLS** | The cluster-agent's Cilium Hubble observability client dials the in-cluster Hubble relay with `insecure.NewCredentials()`, i.e. **unencrypted gRPC**. Scope: pod-to-pod inside one cluster. | `internal/cluster-agent/hubble.go:18-19,63` |
| **OAuth 2.0 / OIDC with PKCE (S256)** | `occ` CLI login | `internal/occ/auth/pkce.go:58-75` |
| **OAuth 2.0 client credentials** | Agent service-to-service tokens, `client_secret_post` | `agents/common/src/common/auth/oauth_client.py:53,89` |
| **JWT / JWS** | Go services accept RSA-signed tokens (RS\*/PS\*, §7); Python agents accept RS\*/ES\*; remote-connect capability tokens are EdDSA | `jwt/middleware.go:100-135`; `agents/.../jwt.py:79`; `remoteconnect/capability.go` |
| **HMAC-SHA256 signatures** | Git provider webhooks | `services/git/signature.go` |
| **X.509 / PKIX, PKCS#1, PKCS#8, PEM** | Certificate and key formats | `remoteconnect.go:1109`; `remoteconnect_provision.go:613-625`; cert-manager |

**APIs used:** Go standard library `crypto/tls`, `crypto/x509`, `crypto/ed25519`, `crypto/ecdsa`, `crypto/rsa`, `crypto/hmac`, `crypto/sha256`, `crypto/rand`; `github.com/golang-jwt/jwt/v5` v5.3.1 (`go.mod:12`), which delegates all cryptography to the standard library; and on the Python side `ssl`, `hashlib`, `pyjwt[crypto]`, `cryptography` and `authlib`.

### 6.1 Single-cluster (k3d) deployment

- **External ingress is plain HTTP.** `gateway.tls.enabled: false` in all three k3d values files (`values-cp.yaml:38-39`, `values-dp.yaml:7-8`, `values-op.yaml:46-47`). The console, API, IdP and Observer are served over `http://…:8080`, `:19080`, `:11080` (`README.md:447-475`), and the OIDC issuer, JWKS and token URLs are `http://` (`values-cp.yaml:28-33`). HTTPS listeners exist when `gateway.tls.enabled: true`, terminated at the gateway with operator-supplied certificates (`templates/gateway/gateway.yaml:32-52`).
- **Inter-plane traffic is encrypted and mutually authenticated:** each agent connects over `wss` with an RSA 2048 client certificate and verifies the gateway against the extracted RSA 4096 `cluster-gateway-ca` (`README.md:140-156`); planes register their agent CA via `clusterAgent.clientCA` (`README.md:216-245`).
- **Internal API mTLS is on by default** (`clusterGateway.internalMtls.enabled: true`, `values.yaml:4061`).
- **Agent services and the console speak plain HTTP** behind the gateway (§1.5).
- **Hubble gRPC is plaintext** in-cluster (`hubble.go:63`).
- **Remote-connect is enabled:** SNI router on NodePort 30443; the chart generates the Ed25519 signing key and each agent certificate is created on first use (`README.md:204-208`).
- **OpenBao runs in dev mode** as the secret backend (`README.md:182-190`); Backstage's backend secret is 32 bytes from `/dev/urandom` (`README.md:104-106`).

---

## (7) How the product prevents user modification of algorithms, key management and key space

The draft answer ("no API or capabilities for the user to modify") is **not accurate**.

**Cannot be modified:**
- **Algorithms, cipher suites and curves.** Compiled into statically linked Go binaries; no config, flag, API or environment variable sets `CipherSuites`, `CurvePreferences` or `MaxVersion`.
- **The accepted JWT algorithm (Go services).** A `SignatureAlgorithm` field exists (`jwt/config.go:68`) and is enforced when set (`jwt/middleware.go:108-115`), but **no caller sets it** (`openchoreo-api/config/security.go:153-165`; `cmd/observer/main.go:758-766`) and no Helm value exposes it. Effective behavior: any algorithm satisfied by the returned `*rsa.PublicKey` is accepted — **RS256/384/512 and PS256/384/512**. The set narrows when the IdP publishes an `alg` on the JWK, since a token whose `alg` disagrees is rejected (`jwks.go:64-65`). Non-RSA JWKS keys are ignored entirely (`jwks.go:101`). So the algorithm is constrained by the IdP, not by OpenChoreo or its operator.
- **The capability-token algorithm**, fixed to EdDSA and enforced with `jwt.WithValidMethods` (`capability.go:16,168,191`).
- **Remote-agent key parameters**, hard-coded to P-256 with a 128-bit serial (`remoteconnect_provision.go:593-597`).
- **No plugin interface or API for supplying cryptographic code.** No REST, MCP or CLI surface exposes a cryptographic operation.

**The TLS floor comes from the library, not from the application.** Most TLS configurations set `MinVersion: tls.VersionTLS12`, but four do not: the JWKS client (`jwt/config.go:101`), the observer authz client (`observer/authz/client.go:46`), the cluster-agent routes (`cluster-agent/router.go:197`), and the API server's `ListenAndServeTLS` (`internal/server/server.go:72`). TLS 1.2 still holds there because it is Go's default (`$GOROOT/src/crypto/tls/common.go:1229-1230`). That default is conditional: `GODEBUG=tls10server=1` re-enables TLS 1.0/1.1 **for servers** that leave `MinVersion` unset, while **clients** remain at 1.2 regardless (the `isClient ||` term in the same condition).

**Operator-configurable (deployment settings, not algorithm changes):**
- **Disabling JWT signature verification entirely (agents).** `JWT_INSECURE_ALLOW_UNVERIFIED` (`observability-plane/templates/sre-agent/rca-agent-config.yaml:25`, `allow_unverified` at `jwt.py:27,30,139-161`) lets an agent run without a JWKS URL and without verifying token signatures. This is the strongest switch in the product and is intended for development only.
- **Disabling TLS verification:** `security.jwksUrlTlsInsecureSkipVerify` (`values.yaml:97`), `uidResolverTlsInsecureSkipVerify` (`:103`), cluster-gateway client `insecure` (`:858`, `:1036`), JWKS `skip_tls_verify` (`:1157`), `remoteConnect.authorizeInsecure` (`:2136`); observability `authzTlsInsecureSkipVerify` (`observability values.yaml:399`) and agent MCP `tlsInsecureSkipVerify` (`:1759`); observer `tls.insecure.skip.verify` (`internal/observer/config/config.go:99`); cluster-agent route `InsecureSkipVerify` (`cluster-agent/router.go:30`); and the agents' `verify_ssl` (`jwt.py:43-45`, `oauth_client.py:90,116,138`).
- **Disabling TLS or mTLS:** `gateway.tls.enabled`, `clusterAgent.tls.enabled`, `clusterGateway.internalMtls.enabled`, and the server's `TLSEnabled` option (`internal/server/server.go:71-75`).
- **Supplying key material:** `clusterGateway.tls.existingSecret` (`values.yaml:4010`), cert-manager `privateKey.algorithm`/`size` in chart templates, `remoteConnect.generateSigningKey: false` with an operator-provided Ed25519 key, and certificate `duration`/`renewBefore`.
- **Go runtime `GODEBUG` settings.** Anyone controlling a pod's environment can shift Go's TLS defaults: `tlsmlkem=0`/`tlssecpmlkem=0` disable the ML-KEM hybrids, `tlsrsakex=1`/`tls3des=1` restore legacy suites, `tlssha1=1` re-enables SHA-1 signatures, `tls10server=1` lowers the server floor (`defaults.go:16-17,58-59`; `common.go:1219,1762`). No chart sets `GODEBUG`.

**Suggested answer:** Users cannot change the cryptographic algorithms, cipher suites or key space. These are fixed in statically compiled binaries and library defaults, and no product API exposes them. Operators can supply their own certificates and keys, set certificate lifetimes, and enable or disable TLS verification and JWT verification through deployment configuration.

---

## (8) Third-party cryptographic components and linking

### 8.1 OpenChoreo's own components

| Component | Language / runtime | Crypto provider | Linking |
|---|---|---|---|
| 9 server binaries — `manager`, `openchoreo-api`, `observer`, `cluster-gateway`, `cluster-agent`, `event-forwarder`, `remote-agent`, `remote-agent-router` (`cmd/`) | Go 1.26.3 (`go.mod:3`) | Go standard library `crypto/*`, pure Go with assembly optimizations. No OpenSSL or BoringCrypto (no `boringcrypto`, `GOFIPS` or `GOEXPERIMENT` anywhere in `make/` or the Dockerfiles). | **Static.** `CGO_ENABLED=0 go build -buildmode=exe` (`make/golang.mk:52-53`), images on `gcr.io/distroless/static:nonroot` (`Dockerfile:3`). |
| `occ` CLI | Go 1.26.3 | Same | **Static binary**, but packaged on `alpine:3.24` rather than distroless, for shell support (`cmd/occ/Dockerfile:2`). |
| sre-agent, finops-agent, portal-assistant (`agents/`) | CPython 3.14, uv-managed build, on `gcr.io/distroless/cc` | **OpenSSL** via Python `ssl`; `cryptography` 50.0.0 (with `cffi` 2.1.1); `pyjwt` 2.13.0; `authlib` 1.7.2; `httpx` 0.28.1; `certifi` 2026.7.22 trust store | **Dynamic.** `distroless/cc` implies dynamically linked glibc, and `cryptography` ships as a prebuilt manylinux/musllinux wheel. **OpenSSL's own linkage and version are not determined from this repository** — open item 1. |
| Console UI (`ghcr.io/openchoreo/openchoreo-ui`, `values.yaml:2811`) | Node.js, built in a separate repository | Node.js / OpenSSL | Not analyzed here — open item 2. |

Other cryptographically relevant direct Go dependencies: `github.com/golang-jwt/jwt/v5` v5.3.1 (`go.mod:12`), `github.com/gorilla/websocket` (`go.mod:16`), and `google.golang.org/grpc` v1.83.0 (`go.mod:159`), used directly by the cluster-agent's Hubble client **without TLS** (`hubble.go:63`).

### 8.2 Third-party products deployed alongside

Separate products installed by the deployment guide. **Their internal cryptography was not analyzed from this repository** and should be described from each vendor's documentation.

| Component | Version | Crypto role | Source |
|---|---|---|---|
| cert-manager (CNCF) | v1.19.4 | Generates RSA keys, issues X.509 certificates | `README.md:38-42` |
| kgateway (Envoy-based) | v2.3.1 | Terminates HTTP/HTTPS ingress when TLS is enabled | `README.md:70-77` |
| Kubernetes Gateway API CRDs | v1.5.1 | Configuration only | `README.md:31-32` |
| External Secrets Operator | 2.0.1 | Syncs secrets into Kubernetes Secrets | `README.md:53-57` |
| **ThunderID** (identity provider) | chart 1.0.1 | Issues OIDC/OAuth2 tokens; publishes the RSA JWKS | `README.md:86-90` |
| OpenBao | dev mode (script-installed) | Secret storage backend; any at-rest encryption belongs to this component | `README.md:182-190` |
| Argo Workflows, twuni docker-registry | chart-managed | Workflow plane | `README.md:264-274` |
| OpenSearch, Prometheus | observability modules | Logs and metrics | `README.md:406-413` |

### 8.3 Suggested text for (8)

> OpenChoreo's server binaries and CLI are written in Go 1.26 and use only the Go standard library cryptographic packages (`crypto/tls`, `crypto/x509`, `crypto/rsa`, `crypto/ecdsa`, `crypto/ed25519`, `crypto/hmac`, `crypto/sha256`, `crypto/rand`), published by The Go Authors (https://go.dev, https://github.com/golang/go/tree/master/src/crypto). `github.com/golang-jwt/jwt/v5` is used for JWT encoding and delegates all cryptographic operations to the standard library. These binaries are built with `CGO_ENABLED=0` and are statically linked; no third-party native cryptographic library is linked into them.
>
> OpenChoreo additionally ships three Python services (sre-agent, finops-agent, portal-assistant) built on CPython 3.14, which use **OpenSSL** through the Python `ssl` module and the `cryptography` package (version 50.0.0) for TLS and JWT signature verification. These components are dynamically linked. The console UI is a Node.js application built in a separate repository. No cryptographic hardware components are used, and certificates are provisioned by cert-manager, a separately installed open-source Kubernetes component.

---

## Open items to confirm before submission

1. **Python agent crypto provenance.** Confirm the OpenSSL version and whether it is statically or dynamically linked, for both the uv-managed CPython 3.14 build (`astral-sh/python-build-standalone`) and the `cryptography` 50.0.0 wheel. Take this from upstream build documentation, not from this repository.
2. **Is `openchoreo-ui` in scope?** It is OpenChoreo's own Backstage-based console, but is built in a separate repository, so its Node.js/OpenSSL cryptography is not covered here.
3. **`SignatureAlgorithm`.** Decide whether to wire this setting up so the accepted JWT algorithm becomes configurable and pinnable, or to answer (7) describing current behavior (algorithm constrained by the identity provider's JWKS).
4. **Scope of the product under classification.** If "Developer Platform" includes the bundled third-party components in §8.2 (kgateway/Envoy, ThunderID, OpenBao, OpenSearch), each has its own cryptography — notably OpenBao for encryption at rest — and must be described from vendor documentation.
5. **Production versus single-cluster.** The single-cluster guide serves external ingress over plain HTTP. Production deployments are expected to set `gateway.tls.enabled: true` with operator-supplied certificates.
