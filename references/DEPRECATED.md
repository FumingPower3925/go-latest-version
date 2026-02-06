# Deprecated Patterns and GODEBUG Reference

This reference lists all deprecated patterns, removed features, and GODEBUG settings
for Go 1.22 through Go 1.26. For the quick reference, see the main [SKILL.md](../SKILL.md).

---

## Table of Contents

- [Deprecated APIs and Their Replacements](#deprecated-apis-and-their-replacements)
- [Removed Features](#removed-features)
- [GODEBUG Settings](#godebug-settings)
- [Breaking Behavioral Changes](#breaking-behavioral-changes)
- [Security-Related Changes](#security-related-changes)
- [Platform Changes](#platform-changes)

---

## Deprecated APIs and Their Replacements

### math/rand (replaced by math/rand/v2, Go 1.22)

| Deprecated | Replacement | Notes |
|---|---|---|
| `math/rand.Seed(n)` | Remove entirely | Auto-seeded since 1.20; no-op since 1.24 |
| `math/rand.Read(b)` | `crypto/rand.Read(b)` | Deprecated since 1.20 |
| `math/rand.Intn(n)` | `math/rand/v2.IntN(n)` | Capital N convention |
| `math/rand.Int31()` | `math/rand/v2.Int32()` | Consistent naming |
| `math/rand.Int31n(n)` | `math/rand/v2.Int32N(n)` | |
| `math/rand.Int63()` | `math/rand/v2.Int64()` | |
| `math/rand.Int63n(n)` | `math/rand/v2.Int64N(n)` | |
| `math/rand.NewSource(seed)` | `rand.NewPCG(s1, s2)` or `rand.NewChaCha8(seed)` | Named generators |
| `math/rand.Source64` interface | Single `Uint64` method on `Source` | No separate interface in v2 |
| `golang.org/x/exp/rand` | `math/rand/v2` | Experimental superseded |

New in v2: `rand.N[T](max)` generic function, `UintN`, `Uint32`, `Uint64`, etc.
Default generator: ChaCha8 (cryptographically strong).

### runtime.SetFinalizer (replaced by AddCleanup, Go 1.24)

| Deprecated | Replacement | Notes |
|---|---|---|
| `runtime.SetFinalizer(obj, fn)` | `runtime.AddCleanup(obj, fn, arg)` | New code should use AddCleanup |

AddCleanup advantages:
- Multiple cleanups per object (SetFinalizer allows only one)
- No object resurrection (cleanup receives arg, not the object)
- Works with interior pointers
- Cycles of objects with cleanups can be GC'd
- Cleaner API with explicit argument passing

### crypto/rsa Encryption (Go 1.26)

| Deprecated | Replacement | Notes |
|---|---|---|
| `rsa.EncryptPKCS1v15(rand, pub, msg)` | `rsa.EncryptOAEP(hash, rand, pub, msg, label)` | PKCS#1 v1.5 is unsafe |
| `rsa.DecryptPKCS1v15(rand, priv, ct)` | `rsa.DecryptOAEP(hash, rand, priv, ct, label)` | |
| `rsa.DecryptPKCS1v15SessionKey(...)` | OAEP-based alternatives | |
| `rsa.GenerateMultiPrimeKey(...)` | `rsa.GenerateKey(nil, bits)` | Multi-prime deprecated |

### crypto/ecdsa Fields (Go 1.26)

| Deprecated | Notes |
|---|---|
| `ecdsa.PublicKey.X`, `ecdsa.PublicKey.Y` (big.Int) | Use alternative representations |
| `ecdsa.PrivateKey.D` (big.Int) | Use alternative representations |

### crypto Random Readers (Go 1.26)

All crypto functions that accepted `io.Reader` for randomness now **ignore** the
parameter and always use the system's secure random source:

```go
// These all ignore the rand parameter -- pass nil
ecdsa.GenerateKey(elliptic.P256(), nil)
rsa.GenerateKey(nil, 2048)
ecdh.P256().GenerateKey(nil)
rand.Prime(nil, 64)
ed25519.GenerateKey(nil)
```

Exception: `ed25519.GenerateKey(rand)` still uses the reader if non-nil.

Use `testing/cryptotest.SetGlobalRandom(t, seed)` for deterministic testing.
Revert with `GODEBUG=cryptocustomrand=1` (temporary, will be removed).

### HTTP

| Deprecated | Replacement | Since |
|---|---|---|
| `httputil.ReverseProxy.Director` | `httputil.ReverseProxy.Rewrite` | 1.26 |

### crypto/cipher

| Deprecated | Replacement | Since |
|---|---|---|
| `cipher.NewOFB` | Modern AEAD modes (GCM) | 1.24 |
| `cipher.NewCFBEncrypter` | Modern AEAD modes (GCM) | 1.24 |
| `cipher.NewCFBDecrypter` | Modern AEAD modes (GCM) | 1.24 |

### Other Deprecations

| Deprecated | Replacement | Since |
|---|---|---|
| `runtime.GOROOT()` | Use `go env GOROOT` or build info | 1.24 |
| `testing/synctest.Run` | `testing/synctest.Test` | 1.25 |
| `go/parser.ParseDir` | -- | 1.25 |
| `go/ast.FilterPackage` | -- | 1.25 |
| `go/ast.PackageExports` | -- | 1.25 |
| `go/ast.MergePackageFiles` | -- | 1.25 |
| `go/ast.MergeMode` type | -- | 1.25 |
| `crypto/elliptic.Inverse` | -- | Removed 1.25 |
| `crypto/elliptic.CombinedMult` | -- | Removed 1.25 |

---

## Removed Features

### Go 1.24
- SHA-1 signature verification in `crypto/x509` (`x509sha1` GODEBUG removed)
- `go get` in legacy GOPATH mode (removed in 1.22)

### Go 1.25
- `crypto/elliptic.Inverse` and `CombinedMult` (undocumented methods)
- `runtimecontentionstacks` GODEBUG setting

### Go 1.26
- `cmd/doc` and `go tool doc` (use `go doc` instead)
- `windows/arm` (32-bit) port
- `signext` and `satconv` GOWASM settings (now unconditional)
- All historical `go fix` fixers (replaced by analysis-based fixers)
- `testing/synctest.Run` (use `Test` instead)

### Scheduled for Removal in Go 1.27
- `GODEBUG=asynctimerchan` (timer channel buffering)
- `GODEBUG=gotypesalias` (type alias behavior)
- `GOEXPERIMENT=nogreenteagc` (Green Tea GC opt-out)
- `GODEBUG=tlsunsafeekm`
- `GODEBUG=tlsrsakex`
- `GODEBUG=tls10server`
- `GODEBUG=tls3des`
- `GODEBUG=x509keypairleaf`
- `GODEBUG=cryptocustomrand`
- `GODEBUG=urlstrictcolons`

---

## GODEBUG Settings

### Go 1.22 Settings

| Setting | Default | Description |
|---|---|---|
| `httpmuxgo121=1` | 0 (new routing) | Revert to Go 1.21 ServeMux routing |
| `tlsmaxrsasize=N` | 8192 | Max RSA key size in TLS handshakes |
| `tls10server=1` | 0 (TLS 1.2 min) | Allow TLS 1.0/1.1 for servers |
| `tlsrsakex=1` | 0 (disabled) | Enable RSA key exchange in TLS |
| `httplaxcontentlength=1` | 0 | Lax Content-Length parsing |
| `disablethp=1` | 0 | Disable transparent huge pages |

### Go 1.23 Settings

| Setting | Default | Description |
|---|---|---|
| `asynctimerchan=1` | 0 (unbuffered) | Timer channel uses old buffered behavior |
| `tls3des=1` | 0 (disabled) | Enable 3DES cipher suites |
| `x509negativeserial=1` | 0 (reject) | Allow negative certificate serial numbers |
| `x509keypairleaf=0` | 1 (populate) | Don't populate X509KeyPair Leaf field |
| `winsymlink=0` | 1 | Disable Windows symlink mode bits |
| `winreadlinkvolume=0` | 1 | Disable Windows Readlink volume normalization |

### Go 1.24 Settings

| Setting | Default | Description |
|---|---|---|
| `fips140=off\|on\|only` | off | FIPS 140-3 mode |
| `randseednop=0` | 1 (no-op) | Make `math/rand.Seed` functional again |
| `rsa1024min=0` | 1 (enforce) | Allow RSA keys < 1024 bits |
| `x509rsacrt=0` | 1 (validate) | Skip CRT parameter validation |
| `x509usepolicies=0` | 1 | Use old `PolicyIdentifiers` field |
| `gotestjsonbuildtext=1` | 0 (JSON) | `go test -json` build error format |
| `multipathtcp=N` | 2 (listeners) | Multipath TCP enablement |
| `tlsmlkem=0` | 1 (enabled) | Disable post-quantum key exchange |

### Go 1.25 Settings

| Setting | Default | Description |
|---|---|---|
| `containermaxprocs=0` | 1 (enabled) | Ignore cgroup CPU limits for GOMAXPROCS |
| `updatemaxprocs=0` | 1 (enabled) | Don't auto-update GOMAXPROCS |
| `tlssha1=1` | 0 (disabled) | Allow SHA-1 in TLS 1.2 handshakes |
| `x509sha256skid=0` | 1 (SHA-256) | Use SHA-1 for SubjectKeyId |
| `allowmultiplevcs=1` | 0 (disabled) | Allow multiple VCS metadata dirs |
| `decoratemappings=0` | 1 (enabled) | Disable OS memory mapping annotations |
| `embedfollowsymlinks=1` | 0 (disabled) | Follow symlinks in embed |

### Go 1.26 Settings

| Setting | Default | Description |
|---|---|---|
| `cryptocustomrand=1` | 0 (ignore) | Honor custom random reader in crypto funcs |
| `httpcookiemaxnum=N` | 3000 | Max cookies per HTTP request |
| `urlmaxqueryparams=N` | 10000 | Max URL query parameters |
| `urlstrictcolons=0` | 1 (strict) | Allow malformed URLs with colons in host |
| `tlssecpmlkem=0` | 1 (enabled) | Disable SecP MLKEM post-quantum KEM |

### GOEXPERIMENT Flags

| Flag | Since | Status | Description |
|---|---|---|---|
| `GOEXPERIMENT=rangefunc` | 1.22 | Stable in 1.23 | Range over function iterators |
| `GOEXPERIMENT=aliastypeparams` | 1.23 | Stable in 1.24 | Generic type aliases |
| `GOEXPERIMENT=synctest` | 1.24 | Stable in 1.25 | testing/synctest package |
| `GOEXPERIMENT=jsonv2` | 1.25 | Experimental | encoding/json/v2 |
| `GOEXPERIMENT=greenteagc` | 1.25 | Default in 1.26 | Green Tea garbage collector |
| `GOEXPERIMENT=nogreenteagc` | 1.26 | Opt-out (removed in 1.27) | Disable Green Tea GC |
| `GOEXPERIMENT=simd` | 1.26 | Experimental | SIMD/vectorized operations (amd64) |
| `GOEXPERIMENT=runtimesecret` | 1.26 | Experimental | secret.Do for forward secrecy |
| `GOEXPERIMENT=goroutineleakprofile` | 1.26 | Experimental | Goroutine leak detection |
| `GOEXPERIMENT=nosizespecializedmalloc` | 1.26 | Opt-out (removed in 1.27) | Disable optimized small allocs |
| `GOEXPERIMENT=norandomizedheapbase64` | 1.26 | Opt-out | Disable heap address randomization |

---

## Breaking Behavioral Changes

### Go 1.22
- **Loop variable scoping**: Each `for` loop iteration creates new variables (controlled by go.mod version).
- **ServeMux routing**: Patterns with `{` and `}` are now interpreted as wildcards. Set `httpmuxgo121=1` to revert.
- **slices.Insert**: Now always panics if index is out of range (even with zero elements to insert).
- **slices.Delete/Compact/Replace**: Now zero elements between new and old length.

### Go 1.23
- **Timer/Ticker channels**: Unbuffered (capacity 0). No stale values after Stop/Reset. Unreferenced timers/tickers are GC-eligible. Controlled by go.mod version.
- **`//go:linkname` restrictions**: Linker disallows referencing unmarked internal stdlib symbols.
- **macOS**: Minimum macOS 11 Big Sur required.

### Go 1.24
- **crypto/x509**: SHA-1 signature verification removed entirely.
- **crypto/rsa**: Keys < 1024 bits rejected by default.
- **math/rand.Seed**: Becomes a no-op globally.
- **Swiss Tables**: Map implementation changed. `reflect.DeepEqual` on `sync.Map` may differ.
- **os.Root**: `Root.Open("../")` initially allowed parent directory access; fixed in 1.24.3.

### Go 1.25
- **SHA-1 in TLS 1.2**: Disabled by default per RFC 9155.
- **SubjectKeyId**: `CreateCertificate` uses SHA-256 instead of SHA-1.
- **testing/synctest**: `Run` function deprecated; use `Test`.
- **testing.AllocsPerRun**: Panics if parallel tests are running.

### Go 1.26
- **image/jpeg**: New encoder/decoder may produce different bit-for-bit output.
- **net/url.Parse**: Rejects malformed URLs with colons in host.
- **Crypto random readers**: Ignored in most crypto functions.
- **net/http cookies**: Use `Request.Host` for scoping.
- **cmd/doc**: Removed (use `go doc`).

---

## Security-Related Changes

### Crypto Minimums
| Requirement | Since | Override |
|---|---|---|
| TLS 1.2 minimum for servers | 1.22 | `tls10server=1` |
| RSA key exchange disabled | 1.22 | `tlsrsakex=1` |
| 3DES cipher suites disabled | 1.23 | `tls3des=1` |
| SHA-1 X.509 signatures removed | 1.24 | None (permanently removed) |
| RSA minimum 1024-bit keys | 1.24 | `rsa1024min=0` |
| SHA-1 in TLS 1.2 handshakes disabled | 1.25 | `tlssha1=1` |
| PKCS#1 v1.5 encryption deprecated | 1.26 | Use OAEP instead |
| Custom crypto randomness ignored | 1.26 | `cryptocustomrand=1` |

### Post-Quantum Cryptography
| Feature | Since | Override |
|---|---|---|
| X25519MLKEM768 in TLS | 1.24 | `tlsmlkem=0` |
| `crypto/mlkem` package (FIPS 203) | 1.24 | -- |
| SecP256r1MLKEM768, SecP384r1MLKEM1024 in TLS | 1.26 | `tlssecpmlkem=0` |
| `crypto/hpke` (RFC 9180) | 1.26 | -- |

### HTTP Cookie Limit (Go 1.26)
`net/http` now limits cookies to 3,000 per request to prevent memory exhaustion
(CVE-2025-58186). Configurable via `GODEBUG=httpcookiemaxnum=N`.

### HTTP URL Query Limit (Go 1.26)
URL query parameter count limited to 10,000 by default.
Configurable via `GODEBUG=urlmaxqueryparams=N`.

---

## Platform Changes

### Go 1.26 Platform Notes
| Platform | Change |
|---|---|
| macOS 12 Monterey | Last supported in Go 1.26. Go 1.27 requires macOS 13+ |
| linux/riscv64 | Race detector now supported |
| s390x | Register-based function calling |
| windows/arm (32-bit) | Removed |
| freebsd/riscv64 | Broken (issue #76475) |
| PowerPC ELFv1 | Last supported in Go 1.26. Go 1.27 switches to ELFv2 |
| WebAssembly | Sign extension and non-trapping float-to-int mandatory |
| windows/arm64 | cgo internal linking supported |

### Minimum Requirements
| Requirement | Version |
|---|---|
| macOS | 11 Big Sur (1.23+), 13 Ventura (1.27+) |
| Linux kernel | 3.2+ (since 1.24) |
| Bootstrap compiler | Go 1.24.6+ (for building 1.26) |

---

## Notable CVEs Fixed in Patch Releases

These are the most impactful security fixes across 1.22.x -- 1.26.x patch releases
that may affect application behavior:

| CVE | Version | Impact |
|---|---|---|
| CVE-2023-45288 | 1.22.2 | HTTP/2 continuation flood (CPU exhaustion) |
| CVE-2024-24790 | 1.22.4 | `netip.Is*` methods wrong for IPv4-mapped IPv6 |
| CVE-2024-24791 | 1.22.5 | HTTP/1.1 Expect: 100-continue denial of service |
| CVE-2024-34156 | 1.22.7/1.23.1 | encoding/gob stack exhaustion |
| CVE-2024-45336 | 1.22.11/1.23.5 | HTTP sensitive header restoration after redirect |
| CVE-2025-22871 | 1.23.8/1.24.2 | HTTP request smuggling via bare LF in chunks |
| CVE-2025-22873 | 1.24.3 | os.Root parent directory escape |
| CVE-2025-4674 | 1.23.11/1.24.5 | VCS command execution in go command |
| CVE-2025-58186 | 1.24.8/1.25.2 | HTTP cookie parsing memory exhaustion |
| CVE-2025-61732 | 1.24.13/1.25.7 | cgo code smuggling via comments |
| CVE-2025-68121 | 1.24.12/1.25.6 | TLS session ticket key reuse |
