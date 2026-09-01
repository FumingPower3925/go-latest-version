# Deprecations, Removals and Compatibility Switches

Everything Go 1.22 -- 1.27 took away, changed under you, or left behind a knob for.
The quick reference lives in [SKILL.md](../SKILL.md).

- [Deprecated APIs](#deprecated-apis)
- [Removed](#removed)
- [GODEBUG settings](#godebug-settings)
- [GOEXPERIMENT flags](#goexperiment-flags)
- [Breaking behaviour changes](#breaking-behaviour-changes)
- [Security defaults](#security-defaults)
- [Platforms and toolchain support](#platforms-and-toolchain-support)

---

## Deprecated APIs

### math/rand -> math/rand/v2 (1.22)

| Deprecated | Replacement |
|---|---|
| `rand.Seed(n)` | Delete it. Auto-seeded since 1.20, a no-op since 1.24 |
| `rand.Read(b)` | `crypto/rand.Read(b)` |
| `rand.Intn(n)` | `rand/v2.IntN(n)` |
| `rand.Int31()` / `Int31n(n)` | `rand/v2.Int32()` / `Int32N(n)` |
| `rand.Int63()` / `Int63n(n)` | `rand/v2.Int64()` / `Int64N(n)` |
| `rand.NewSource(seed)` | `rand/v2.NewPCG(s1, s2)` or `NewChaCha8(seed)` |
| `rand.Source64` | `rand/v2.Source` (one `Uint64` method) |
| `golang.org/x/exp/rand` | `math/rand/v2` |

v2 adds the generic `rand.N[T](max)` (and, since 1.27, `(*Rand).N`). The default generator
is ChaCha8.

### runtime.SetFinalizer -> runtime.AddCleanup (1.24)

`AddCleanup` allows many cleanups per object, never resurrects the object, works with
interior pointers, and lets cycles of cleanup-bearing objects be collected. `SetFinalizer`
does none of that.

### crypto (1.26 / 1.27)

| Deprecated | Replacement | Since |
|---|---|---|
| `rsa.EncryptPKCS1v15` / `DecryptPKCS1v15` / `DecryptPKCS1v15SessionKey` | `rsa.EncryptOAEP` / `DecryptOAEP` | 1.26 |
| `rsa.GenerateMultiPrimeKey` | `rsa.GenerateKey(nil, bits)` | 1.26 |
| `ecdsa.PublicKey.X` / `.Y`, `ecdsa.PrivateKey.D` (`big.Int`) | `Bytes()` / `NewPublicKey` style APIs | 1.26 |
| `tls.Config.Rand` | `cryptotest.SetGlobalRandom` for tests; nothing in production | 1.27 |
| `cipher.NewOFB`, `NewCFBEncrypter`, `NewCFBDecrypter` | AEAD modes (`cipher.NewGCMWithRandomNonce`) | 1.24 |

Since 1.26 every crypto function that took an `io.Reader` for randomness **ignores** it and
uses the system CSPRNG. Pass `nil`:

```go
ecdsa.GenerateKey(elliptic.P256(), nil)
rsa.GenerateKey(nil, 2048)
ecdh.P256().GenerateKey(nil)
rand.Prime(nil, 64)
```

`ed25519.GenerateKey(rand)` is the exception and still honours a non-nil reader.
`GODEBUG=cryptocustomrand=1` restores the old behaviour temporarily.

### Everything else

| Deprecated | Replacement | Since |
|---|---|---|
| `httputil.ReverseProxy.Director` | `ReverseProxy.Rewrite` | 1.26 |
| `testing/synctest.Run` | `synctest.Test` (removed in 1.26) | 1.25 |
| `runtime.GOROOT()` | `go env GOROOT` or build info | 1.24 |
| `go/parser.ParseDir` | Walk the directory yourself | 1.25 |
| `go/ast.FilterPackage`, `PackageExports`, `MergePackageFiles`, `MergeMode` | -- | 1.25 |

---

## Removed

| Release | Removed |
|---|---|
| 1.24 | SHA-1 signature verification in `crypto/x509` (and `GODEBUG=x509sha1`) |
| 1.25 | `crypto/elliptic.Inverse`, `CombinedMult`; `GODEBUG=runtimecontentionstacks` |
| 1.26 | `cmd/doc` / `go tool doc` (use `go doc`); `testing/synctest.Run`; `windows/arm` (32-bit); `signext` and `satconv` GOWASM settings; all historical `go fix` fixers |
| 1.27 | `bzr` support in the `go` command; `GODEBUG` `asynctimerchan`, `gotypesalias`, `tlsunsafeekm`, `tlsrsakex`, `tls3des`, `tls10server`, `x509keypairleaf`; `GOEXPERIMENT=goroutineleakprofile` (the profile is now GA); the `fmtappendf` `go fix` analyzer; `go fix -waitgroup` (renamed `-waitgroupgo`) |

Announced for later removal:

| Setting | Removal |
|---|---|
| `GOEXPERIMENT=nosizespecializedmalloc` | 1.28 |
| `GODEBUG=gotestjsonbuildtext` | 1.28 at the earliest |
| `GODEBUG=x509sslcertoverrideplatform` | 1.31 |
| `GODEBUG=fips140ems` | 1.31 |
| `GOEXPERIMENT=nojsonv2` | unscheduled; file an issue instead of relying on it |
| `GODEBUG=cryptocustomrand` | unscheduled, but temporary by design |

Since 1.27 the `go` command still **accepts** a removed GODEBUG in `go.mod` (`godebug`) or a
`//go:debug` comment as long as it is set to the value that was the default when it was
removed. Setting it to the old value is a build error.

---

## GODEBUG settings

Defaults below are what Go 1.27 uses. All are settable through the `GODEBUG` environment
variable, a `godebug` line in `go.mod`, or a `//go:debug` comment.

### HTTP, URL and templates

| Setting | Default | Effect of the non-default value |
|---|---|---|
| `httpmuxgo121` | `0` | `1` restores Go 1.21 `ServeMux` (no methods or wildcards) |
| `httplaxcontentlength` | `0` | `1` accepts an empty `Content-Length` header |
| `httpservecontentkeepheaders` | `0` | `1` keeps caching headers when `ServeContent` serves an error |
| `httpcookiemaxnum` | `3000` | Max cookies parsed per request; `0` = unlimited (1.26) |
| `http2server` / `http2client` / `http2debug` | on | Disable or trace built-in HTTP/2 |
| `urlmaxqueryparams` | `10000` | Max query parameters; `0` = unlimited (1.26) |
| `urlstrictcolons` | `1` | `0` re-allows `http://localhost:1:2` style hosts (1.26) |
| `htmlmetacontenturlescape` | `1` | `0` stops escaping URLs in `<meta content=...>` (1.27; backported to 1.25.8 / 1.26.1) |
| `multipartmaxheaders` / `multipartmaxparts` | limited | Raise or remove MIME limits |

### Crypto and TLS

| Setting | Default | Effect of the non-default value |
|---|---|---|
| `fips140` | `off` | `on` / `only` for FIPS 140-3 mode (fixed at startup) |
| `fips140ems` | `1` | `0` disables Extended Master Secret enforcement in FIPS mode (1.27) |
| `cryptocustomrand` | `0` | `1` honours custom `io.Reader` randomness again (1.26) |
| `rsa1024min` | `1` | `0` allows RSA keys under 1024 bits (1.24) |
| `tlsmaxrsasize` | `8192` | Max RSA key size accepted in a handshake |
| `tlsmlkem` | `1` | `0` drops X25519MLKEM768 from the default curves (1.24) |
| `tlssecpmlkem` | `1` | `0` drops the SecP MLKEM hybrids (1.26) |
| `tlssha1` | `0` | `1` re-enables SHA-1 in TLS 1.2 handshakes (1.25) |
| `dataindependenttiming` | `0` | `1` enables DIT mode program-wide (arm64) |
| `x509negativeserial` | `0` | `1` accepts negative certificate serial numbers |
| `x509rsacrt` | `1` | `0` skips CRT parameter validation |
| `x509sha256skid` | `1` | `0` reverts `SubjectKeyId` to SHA-1 |
| `x509usepolicies` | `1` | `0` marshals from `PolicyIdentifiers` |
| `x509usefallbackroots` | on | Fallback root behaviour |
| `x509sslcertoverrideplatform` | `1` | `0` ignores `SSL_CERT_FILE`/`SSL_CERT_DIR` on Windows/Darwin (1.27) |

Note the 1.27 default: when `SSL_CERT_FILE` or `SSL_CERT_DIR` is set on Windows or macOS,
`SystemCertPool` loads roots from disk and switches to the pure-Go verifier instead of the
platform APIs.

### Runtime and toolchain

| Setting | Default | Effect of the non-default value |
|---|---|---|
| `containermaxprocs` | `1` | `0` ignores cgroup CPU limits when choosing `GOMAXPROCS` (1.25) |
| `updatemaxprocs` | `1` | `0` stops periodic `GOMAXPROCS` refresh (1.25) |
| `decoratemappings` | `1` | `0` drops `[anon: Go: ...]` annotations in `/proc` maps (1.25) |
| `tracebacklabels` | `1` | `0` omits pprof goroutine labels from traceback headers (default flipped in 1.27) |
| `panicnil` | `0` | `1` restores `panic(nil)` as a nil panic |
| `randseednop` | `1` | `0` makes `math/rand.Seed` functional again (1.24) |
| `randautoseed` | on | Global `math/rand` auto-seeding |
| `execerrdot` | on | Reject `PATH` lookups resolving into the current directory |
| `gotestjsonbuildtext` | `0` | `1` emits build errors as text in `go test -json` |
| `allowmultiplevcs` | `0` | `1` stamps build info when several VCS dirs are present |
| `embedfollowsymlinks` | `0` | `1` lets `//go:embed` follow symlinks (1.25) |
| `installgoroot`, `gocacheverify`, `gocachehash`, `gocachetest` | -- | Build/cache debugging |
| `tarinsecurepath` / `zipinsecurepath` | `1` | `0` rejects insecure archive paths |

### OS and net

| Setting | Default | Effect of the non-default value |
|---|---|---|
| `multipathtcp` | `2` | MPTCP: `0` off, `1` both, `2` listeners, `3` dialers (1.24) |
| `netdns` | auto | Force the pure-Go or cgo resolver |
| `netedns0` | `1` | `0` stops sending EDNS0 headers |
| `winsymlink` | `1` | `0` treats Windows mount points as symlinks again (1.23) |
| `winreadlinkvolume` | `1` | `0` normalises volumes to drive letters again (1.23) |

---

## GOEXPERIMENT flags

Enabled by default in Go 1.27 -- the listed opt-out is temporary:

| Experiment | Default since | Opt-out |
|---|---|---|
| `greenteagc` | 1.26 | `GOEXPERIMENT=nogreenteagc` |
| `jsonv2` | 1.27 | `GOEXPERIMENT=nojsonv2` |
| `sizespecializedmalloc` | 1.27 | `GOEXPERIMENT=nosizespecializedmalloc` (goes away in 1.28) |
| `randomizedheapbase64` | 1.26 | `GOEXPERIMENT=norandomizedheapbase64` |

Opt-in and still experimental in 1.27:

| Experiment | Enables |
|---|---|
| `simd` | The portable `simd` package and architecture-specific `simd/archsimd` (amd64, arm64 Neon, wasm) |
| `runtimesecret` | `runtime/secret`; secret mode now propagates to goroutines started inside it |
| `runtimefreegc` | More eager memory reuse with compiler assistance |
| `mapsplitgroup` | Split key/elem arrays in map groups instead of interleaved slots |
| `arenas`, `cgocheck2`, `newinliner`, `boringcrypto` | Long-standing niche experiments |

Graduated (the flag no longer exists): `rangefunc` (1.23), `aliastypeparams` (1.24),
`synctest` (1.25), `goroutineleakprofile` (1.27).

---

## Breaking behaviour changes

**1.22** -- Loop variables are per-iteration (gated on the `go` directive).
`ServeMux` reads `{`/`}` as wildcards. `slices.Insert` always panics on an out-of-range
index. `slices.Delete`/`Compact`/`Replace` zero the vacated tail.

**1.23** -- Timer and Ticker channels are unbuffered, with no stale value after `Stop`/
`Reset`, and unreferenced timers are collectable (gated on the `go` directive).
`//go:linkname` can no longer reach unmarked internal stdlib symbols. macOS 11 minimum.

**1.24** -- SHA-1 certificate signatures no longer verify at all. RSA keys under 1024 bits
are rejected. `math/rand.Seed` becomes a global no-op. Maps switch to Swiss Tables, so
`reflect.DeepEqual` on a `sync.Map` may differ.

**1.25** -- SHA-1 disabled in TLS 1.2 handshakes (RFC 9155). `CreateCertificate` fills
`SubjectKeyId` with SHA-256. `testing.AllocsPerRun` panics while parallel tests run.

**1.26** -- `image/jpeg` output is not bit-for-bit identical. `net/url.Parse` rejects
malformed colons in the host. Crypto randomness readers are ignored. `cmd/doc` is gone.

**1.27** -- `compress/flate` produces different bytes, so `gzip`, `zlib`, `zip` and `png`
goldens may need regenerating; `unicode` jumps from 15 to 17 for the same reason.
HTTP/1 `Response.Body` drains on `Close` (<=256 KiB, <=50 ms). HTTP/2 servers honour RFC
9218 client priorities (`Server.DisableClientPriority` reverts). Requests are limited to
`DefaultMaxHeaderValueCount` (500) header values. `net.UnixConn` reads return bare
`io.EOF` instead of a wrapped `*net.OpError`. Function-literal symbol names changed and may
now be shared, so comparing function code pointers is even less reliable. Relative `//line`
paths resolve against the containing file's directory.

---

## Security defaults

| Requirement | Since | Override |
|---|---|---|
| TLS 1.2 minimum for servers | 1.22 | none since 1.27 (`tls10server` removed) |
| RSA key exchange disabled | 1.22 | none since 1.27 (`tlsrsakex` removed) |
| 3DES suites disabled | 1.23 | none since 1.27 (`tls3des` removed) |
| SHA-1 X.509 signatures rejected | 1.24 | none, permanently |
| RSA keys >= 1024 bits | 1.24 | `rsa1024min=0` |
| SHA-1 disabled in TLS 1.2 | 1.25 | `tlssha1=1` |
| PKCS#1 v1.5 encryption deprecated | 1.26 | use OAEP |
| Custom crypto randomness ignored | 1.26 | `cryptocustomrand=1` |
| Cookies capped at 3000/request | 1.26 | `httpcookiemaxnum` |
| Query parameters capped at 10000 | 1.26 | `urlmaxqueryparams` |
| Header values capped at 500/request | 1.27 | `Server.MaxHeaderValueCount` |
| `<meta content=...>` URLs escaped | 1.27 | `htmlmetacontenturlescape=0` |

Post-quantum status: `crypto/mlkem` (FIPS 203) since 1.24 with X25519MLKEM768 on by
default; SecP256r1MLKEM768 and SecP384r1MLKEM1024 on by default since 1.26; `MLKEM1024`
opt-in via `Config.CurvePreferences` and `crypto/mldsa` (FIPS 204) signatures, including
TLS 1.3 `MLDSA44/65/87`, since 1.27. From 1.27 the hybrids can be requested explicitly in
`CurvePreferences` even when `tlsmlkem=0` or `tlssecpmlkem=0`.

---

## Platforms and toolchain support

| Requirement | Version |
|---|---|
| macOS | 12 Monterey through 1.26 (its last release), **13 Ventura from 1.27** |
| Linux kernel | 3.2+ (1.24+); 3.13+ for `linux/ppc64` in 1.27 |
| Bootstrap toolchain | Go 1.24.6+ builds 1.26 and 1.27 |

1.27 platform notes: `linux/ppc64` (big-endian) switches to the ELFv2 ABI and gains cgo,
PIE and external linking; `windows/arm` (32-bit) was removed in 1.26; `linux/riscv64`
supports the race detector and s390x uses register-based calls since 1.26.

### Support policy

Only the two most recent major releases get security fixes. As of the Go 1.27.0 release
(2026-08-19) that means **1.27.x and 1.26.x**; 1.25 and older are end of life.

Minimum versions worth pinning today: **go1.27.0**, or **go1.26.7** on the previous line.
Go shipped 2026 stdlib fixes for `os.Root` escapes (CVE-2026-32282, CVE-2026-39822),
`html/template` escaper bypasses (CVE-2026-32289, CVE-2026-39823, CVE-2026-39826,
CVE-2026-56858), `crypto/x509` verification bypasses (CVE-2026-33810), compiler
miscompilations that broke memory safety (CVE-2026-27143, CVE-2026-27144), and checksum
database bypasses in `cmd/go` (CVE-2026-42501, CVE-2026-56864, CVE-2026-56865).

Do not enumerate CVEs by hand -- run `govulncheck ./...`, which reports only the ones your
code actually reaches.
