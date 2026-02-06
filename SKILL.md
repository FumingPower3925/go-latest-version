---
name: go-latest-version
description: >
  Enforces modern Go standards and idioms for Go 1.22 through 1.26. Use when writing,
  reviewing, or modifying Go code to ensure the latest language features, standard library
  APIs, and best practices are used. Prevents deprecated patterns and guides toward
  idiomatic modern Go.
license: Apache-2.0
metadata:
  author: FumingPower3925
  version: "1.0"
  go-versions: "1.22, 1.23, 1.24, 1.25, 1.26"
---

# Modern Go Standards (1.22 -- 1.26)

You are writing Go code targeting the latest Go versions. Always prefer the most modern
idiomatic pattern. If the project's `go.mod` specifies a version, respect that as the
minimum feature set. When no version is specified, assume Go 1.26.

For even more detailed examples, see [MODERN-PATTERNS.md](references/MODERN-PATTERNS.md).
For full deprecation lists and GODEBUG settings, see [DEPRECATED.md](references/DEPRECATED.md).

---

## Critical Rules

1. **Never use deprecated APIs** when a modern replacement exists.
2. **Check go.mod version** before using version-gated features.
3. **Use `range` over integers** instead of C-style `for` loops (Go 1.22+).
4. **Use iterators** (`iter.Seq`, `iter.Seq2`) for sequences (Go 1.23+).
5. **Use `errors.AsType[T]`** instead of `errors.As` (Go 1.26+).
6. **Use `new(expr)`** for inline pointer creation (Go 1.26+).
7. **Use `b.Loop()`** instead of `for range b.N` in benchmarks (Go 1.24+).
8. **Use `t.Context()`** instead of manually creating cancelable contexts in tests (Go 1.24+).
9. **Use `sync.WaitGroup.Go()`** instead of manual `Add`/`Done` patterns (Go 1.25+).
10. **Use `crypto/rand.Text()`** for random tokens instead of custom generators (Go 1.24+).
11. **Pass `nil` for the `rand` parameter** in crypto functions (Go 1.26+).
12. **Use `runtime.AddCleanup`** instead of `runtime.SetFinalizer` (Go 1.24+).
13. **Use `omitzero`** instead of `omitempty` for struct fields in JSON tags (Go 1.24+).
14. **Use `math/rand/v2`** instead of `math/rand` (Go 1.22+).
15. **Use `os.Root`** for directory-scoped filesystem access (Go 1.24+).
16. **Remove loop variable capture hacks** (`v := v`) -- per-iteration scoping is automatic (Go 1.22+).

---

## Do This, Not That

### Language & Syntax

| Instead of (old) | Use (modern) | Since |
|---|---|---|
| `for i := 0; i < n; i++` | `for i := range n` | 1.22 |
| `v := v` in loop closures | Remove it; loop vars are per-iteration | 1.22 |
| `p := 42; foo(&p)` | `foo(new(42))` | 1.26 |
| `errors.As(err, &target)` | `errors.AsType[*T](err)` | 1.26 |
| Custom iterator callbacks | `iter.Seq[V]` / `iter.Seq2[K, V]` with `for range` | 1.23 |

### Standard Library

| Instead of (old) | Use (modern) | Since |
|---|---|---|
| `math/rand.Intn(n)` | `math/rand/v2.IntN(n)` | 1.22 |
| `math/rand.Seed(...)` | Remove it; auto-seeded since 1.20, no-op since 1.24 | 1.22 |
| `math/rand.Int31()` / `Int63()` | `math/rand/v2.Int32()` / `Int64()` | 1.22 |
| `math/rand.Read(b)` | `crypto/rand.Read(b)` | 1.22 |
| `rand.NewSource(seed)` | `rand.NewPCG(s1, s2)` or `rand.NewChaCha8(seed)` | 1.22 |
| `fmt.Sprintf` for errors | `fmt.Errorf` (now equally efficient) | 1.26 |
| `runtime.SetFinalizer` | `runtime.AddCleanup` | 1.24 |
| `sort.Slice(s, less)` | `slices.SortFunc(s, cmp)` | 1.23 |
| `strings.Split` in `for range` | `strings.SplitSeq` | 1.24 |
| `strings.Fields` in `for range` | `strings.FieldsSeq` | 1.24 |
| Manual contains loop | `slices.Contains(s, v)` | 1.21 |
| `io.Discard` with `slog.NewTextHandler` | `slog.DiscardHandler` | 1.24 |
| `slog.Group("k", attrs...)` with `[]slog.Attr` | `slog.GroupAttrs("k", attrs...)` | 1.25 |
| Single `slog` handler | `slog.NewMultiHandler(h1, h2)` for fan-out | 1.26 |

### Error Handling

| Instead of (old) | Use (modern) | Since |
|---|---|---|
| `var target *T; errors.As(err, &target)` | `target, ok := errors.AsType[*T](err)` | 1.26 |
| `errors.New("x")` vs `fmt.Errorf("x")` debate | Use either; `fmt.Errorf` now matches `errors.New` perf | 1.26 |

### HTTP

| Instead of (old) | Use (modern) | Since |
|---|---|---|
| Third-party router for method matching | `mux.HandleFunc("POST /path", h)` | 1.22 |
| Third-party router for path params | `/items/{id}` with `r.PathValue("id")` | 1.22 |
| Custom CSRF middleware | `http.CrossOriginProtection` | 1.25 |
| `httputil.ReverseProxy.Director` | `httputil.ReverseProxy.Rewrite` | 1.26 |
| Manual HTTP protocol negotiation | `Server.Protocols` / `Transport.Protocols` | 1.24 |

### Testing

| Instead of (old) | Use (modern) | Since |
|---|---|---|
| `for range b.N` | `for b.Loop()` | 1.24 |
| `ctx, cancel := context.WithCancel(...)` in tests | `t.Context()` | 1.24 |
| Manual working directory changes in tests | `t.Chdir(dir)` | 1.24 |
| `time.Sleep` in concurrent tests | `testing/synctest.Test` with fake clock | 1.25 |
| `testing/synctest.Run` | `testing/synctest.Test` | 1.25 |
| `sink` variable to prevent optimization | `b.Loop()` keeps values alive | 1.24 |

### Crypto

| Instead of (old) | Use (modern) | Since |
|---|---|---|
| Custom random token generation | `crypto/rand.Text()` | 1.24 |
| `ecdsa.GenerateKey(curve, rand.Reader)` | `ecdsa.GenerateKey(curve, nil)` | 1.26 |
| `rsa.GenerateKey(rand.Reader, bits)` | `rsa.GenerateKey(nil, bits)` | 1.26 |
| `golang.org/x/crypto/sha3` | `crypto/sha3` | 1.24 |
| `golang.org/x/crypto/hkdf` | `crypto/hkdf` | 1.24 |
| `golang.org/x/crypto/pbkdf2` | `crypto/pbkdf2` | 1.24 |
| `EncryptPKCS1v15` | `EncryptOAEP` | 1.26 |
| `cipher.NewGCM` + manual nonce | `cipher.NewGCMWithRandomNonce` | 1.24 |
| SHA-1 certificates | SHA-256+ certificates (SHA-1 removed in 1.24) | 1.24 |
| RSA keys < 1024 bits | RSA keys >= 2048 bits | 1.24 |

### Filesystem

| Instead of (old) | Use (modern) | Since |
|---|---|---|
| `os.Open` with path validation | `os.OpenRoot(dir)` + `root.Open(file)` | 1.24 |
| Manual recursive directory copy | `os.CopyFS(dst, srcFS)` | 1.23 |

### Concurrency

| Instead of (old) | Use (modern) | Since |
|---|---|---|
| `wg.Add(1); go func() { defer wg.Done(); f() }()` | `wg.Go(f)` | 1.25 |
| String deduplication via maps | `unique.Make(s)` for canonical handles | 1.23 |
| Timer drain before Reset | `t.Stop(); t.Reset(d)` -- no drain needed | 1.23 |
| `time.After` in loops | Still works (GC-safe since 1.23), but `NewTimer` + `Reset` is better | 1.23 |

---

## Inline Code Examples

These examples cover the most common patterns. The agent should use these patterns
whenever writing Go code, without needing to consult reference files.

### Range over Integers (Go 1.22+)

```go
// OLD: for i := 0; i < n; i++ { process(i) }
// NEW:
for i := range n {
    process(i)
}
```

### Loop Variables Are Per-Iteration (Go 1.22+)

```go
// This is safe now -- each iteration gets its own `v`:
for _, v := range values {
    go func() {
        fmt.Println(v) // unique per iteration, no data race
    }()
}
// DELETE any `v := v` capture hacks -- they are no longer needed.
```

### new(expr) for Pointer Fields (Go 1.26+)

```go
type Config struct {
    Port    *int    `json:"port,omitzero"`
    Debug   *bool   `json:"debug,omitzero"`
    Timeout *string `json:"timeout,omitzero"`
}

cfg := Config{
    Port:    new(8080),
    Debug:   new(true),
    Timeout: new("30s"),
}
```

### errors.AsType (Go 1.26+)

```go
// OLD:
// var pathErr *fs.PathError
// if errors.As(err, &pathErr) { ... }

// NEW -- type-safe, no reflection, no panic risk:
if pathErr, ok := errors.AsType[*fs.PathError](err); ok {
    fmt.Println("path:", pathErr.Path)
}

// Multiple error types -- variables scoped to their blocks:
if connErr, ok := errors.AsType[*net.OpError](err); ok {
    handleNetworkError(connErr)
} else if dnsErr, ok := errors.AsType[*net.DNSError](err); ok {
    handleDNSError(dnsErr)
}
```

### Iterators (Go 1.23+)

```go
// Define a single-value iterator:
func Reversed[V any](s []V) iter.Seq[V] {
    return func(yield func(V) bool) {
        for i := len(s) - 1; i >= 0; i-- {
            if !yield(s[i]) {
                return
            }
        }
    }
}

// Use with range:
for v := range Reversed(mySlice) {
    fmt.Println(v)
}

// Standard library iterators:
for v := range slices.Values(s) { }           // values only
for i, v := range slices.Backward(s) { }      // reverse
for chunk := range slices.Chunk(s, 3) { }     // chunked
for k := range maps.Keys(m) { }               // map keys
s2 := slices.Collect(slices.Values(s))         // collect into slice
s2 := slices.Sorted(slices.Values(s))          // collect + sort

// Lazy string iteration (Go 1.24+):
for line := range strings.Lines(text) { }         // line-by-line
for part := range strings.SplitSeq(s, ",") { }    // lazy split
for word := range strings.FieldsSeq(s) { }        // whitespace split
```

### math/rand/v2 (Go 1.22+)

```go
import "math/rand/v2"

n := rand.IntN(100)           // random int in [0, 100)
n := rand.N(max)              // generic -- works with any integer type
d := rand.N(100*time.Millisecond) // works with time.Duration too
u := rand.Uint64()            // random uint64
// Default generator is ChaCha8 (cryptographically strong)
// No need to seed -- auto-seeded
```

### HTTP Routing (Go 1.22+)

```go
mux := http.NewServeMux()
mux.HandleFunc("GET /api/users", listUsers)
mux.HandleFunc("POST /api/users", createUser)
mux.HandleFunc("GET /api/users/{id}", func(w http.ResponseWriter, r *http.Request) {
    id := r.PathValue("id")
    user, err := db.GetUser(id)
    // ...
})
mux.HandleFunc("GET /static/{path...}", func(w http.ResponseWriter, r *http.Request) {
    path := r.PathValue("path") // matches entire remaining path
    http.ServeFile(w, r, filepath.Join("static", path))
})
```

### CSRF Protection (Go 1.25+)

```go
mux := http.NewServeMux()
// ... register handlers ...

protection := http.NewCrossOriginProtection()
protection.AddTrustedOrigin("https://myapp.example.com")
http.ListenAndServe(":8080", protection.Handler(mux))
// Safe methods (GET, HEAD, OPTIONS) always pass through.
```

### Testing: b.Loop() (Go 1.24+)

```go
func BenchmarkProcess(b *testing.B) {
    data := expensiveSetup() // runs once, not b.N times
    for b.Loop() {           // no b.ResetTimer, no sink variable
        process(data)        // compiler won't optimize away
    }
}
```

### Testing: t.Context() (Go 1.24+)

```go
func TestServer(t *testing.T) {
    srv := startServer(t.Context()) // canceled after test, before cleanup
    t.Cleanup(func() {
        <-srv.Done() // wait for graceful shutdown
    })
    // ... test srv ...
}
```

### Testing: synctest.Test (Go 1.25+)

```go
import "testing/synctest"

func TestTimeout(t *testing.T) {
    synctest.Test(t, func(t *testing.T) {
        // Fake clock: time advances when all goroutines block
        ch := make(chan int)
        _, err := ReadWithTimeout(ch, time.Minute) // resolves instantly
        if err == nil {
            t.Fatal("expected timeout")
        }
    })
}
// Do NOT call t.Run, t.Parallel, or t.Deadline inside the bubble.
```

### Testing: t.Chdir, t.Attr, t.ArtifactDir

```go
func TestFileOps(t *testing.T) {
    t.Chdir(t.TempDir()) // restored after test (Go 1.24+)
    os.WriteFile("test.txt", []byte("hello"), 0o644)
}

func TestFeature(t *testing.T) {
    t.Attr("team", "platform")        // metadata (Go 1.25+)
    t.Attr("issue", "PROJ-1234")
    dir := t.ArtifactDir()            // artifact output (Go 1.26+)
    os.WriteFile(filepath.Join(dir, "debug.log"), logData, 0o644)
}
```

### WaitGroup.Go (Go 1.25+)

```go
var wg sync.WaitGroup
wg.Go(func() { result1 = fetchFromAPI() })
wg.Go(func() { result2 = queryDatabase() })
wg.Wait()
```

### unique Package (Go 1.23+)

```go
import "unique"

type Server struct {
    Zone unique.Handle[string]
}
func NewServer(zone string) Server {
    return Server{Zone: unique.Make(zone)} // canonical reference
}
// Comparison is pointer-level fast: s1.Zone == s2.Zone
```

### os.Root -- Safe Filesystem (Go 1.24+)

```go
root, err := os.OpenRoot("/var/data")
if err != nil { return err }
defer root.Close()

data, _ := root.ReadFile("config.json")      // scoped to /var/data
root.WriteFile("out.json", data, 0o644)       // safe
_, err = root.Open("../etc/passwd")            // ERROR: escapes root
root.MkdirAll("a/b/c", 0o755)                 // Go 1.25+
root.Rename("old.txt", "new.txt")             // Go 1.25+
root.RemoveAll("temp")                        // Go 1.25+
```

### Crypto Patterns (Go 1.24+/1.26+)

```go
// Random token (Go 1.24+):
token := crypto_rand.Text() // base32, 128+ bits

// Key generation -- pass nil for rand (Go 1.26+):
ecKey, _ := ecdsa.GenerateKey(elliptic.P256(), nil)
rsaKey, _ := rsa.GenerateKey(nil, 2048)

// SHA-3 (Go 1.24+):
hash := sha3.Sum256(data)

// GCM with auto nonce (Go 1.24+):
block, _ := aes.NewCipher(key)
aead, _ := cipher.NewGCMWithRandomNonce(block)
ct := aead.Seal(nil, nil, plaintext, aad) // nonce auto-prepended

// HKDF / PBKDF2 now in stdlib (Go 1.24+):
dk := hkdf.Key(sha256.New, secret, salt, info, 32)
dk := pbkdf2.Key([]byte(password), salt, 600000, 32, sha256.New)
```

### JSON: omitzero (Go 1.24+)

```go
type Event struct {
    Name  string    `json:"name"`
    Start time.Time `json:"start,omitzero"` // correctly omits zero time.Time
    End   *Info     `json:"end,omitzero"`   // omits nil pointer
}
// omitzero uses IsZero() when available; omitempty uses legacy rules.
// Prefer omitzero for struct types.
```

### Logging (Go 1.24+/1.25+/1.26+)

```go
// Discard handler (Go 1.24+):
logger := slog.New(slog.DiscardHandler)

// Multi-handler fan-out (Go 1.26+):
logger := slog.New(slog.NewMultiHandler(
    slog.NewJSONHandler(os.Stdout, nil),
    slog.NewTextHandler(logFile, nil),
))

// GroupAttrs from a slice (Go 1.25+):
attrs := []slog.Attr{slog.String("method", "GET"), slog.Int("status", 200)}
logger.Info("request", slog.GroupAttrs("http", attrs...))
```

### Timer/Ticker (Go 1.23+)

```go
// Unreferenced timers are GC'd -- safe in select loops:
select {
case v := <-ch:
    process(v)
case <-time.After(timeout):
    log.Warn("timeout")
}

// No drain before Reset (Go 1.23+):
t := time.NewTimer(d)
t.Stop()
t.Reset(newDuration)
<-t.C
```

### Reflect Iterators (Go 1.26+)

```go
typ := reflect.TypeFor[http.Client]()
for f := range typ.Fields() {
    fmt.Println(f.Name, f.Type)
}
for m := range typ.Methods() {
    fmt.Println(m.Name, m.Type)
}
```

### Tool Directives in go.mod (Go 1.24+)

```bash
go get -tool golang.org/x/tools/cmd/stringer
go tool stringer -type=Color
# No more tools.go with blank imports needed.
```

### go fix Modernization (Go 1.26+)

```bash
go fix ./...          # modernize all code
go fix -diff ./...    # preview changes without applying
go fix -rangeint .    # only specific fixer
```

Fixers include: `rangeint`, `bloop`, `waitgroup`, `omitzero`, `slicescontains`,
`slicessort`, `minmax`, `stringscut`, `stringsseq`, `forvar`, `newexpr`.

---

## Feature Availability by Version

### Go 1.22
- Per-iteration loop variable scoping
- `for i := range n` (range over integers)
- `math/rand/v2` package
- Enhanced `net/http.ServeMux` routing (methods, wildcards, `{param}`, `{path...}`, `{$}`)
- `slices.Concat`, zeroing behavior for `slices.Delete`/`Compact`/`Replace`
- `go/version` package
- `database/sql.Null[T]` generic type
- Range-over-func preview (`GOEXPERIMENT=rangefunc`)

### Go 1.23
- Range over function iterators (`iter.Seq`, `iter.Seq2`, `iter.Pull`)
- `slices.All`, `Values`, `Backward`, `Collect`, `AppendSeq`, `Sorted`, `SortedFunc`, `Chunk`
- `maps.All`, `Keys`, `Values`, `Insert`, `Collect`
- `unique` package (value interning/canonicalization)
- Timer/Ticker: GC-eligible without Stop; no stale values after Reset/Stop
- `os.CopyFS`
- `http.ParseCookie`, `http.ParseSetCookie`, `Cookie.Partitioned`
- `slices.Repeat`
- `structs` package (`structs.HostLayout`)

### Go 1.24
- Generic type aliases
- `weak` package (weak pointers)
- `runtime.AddCleanup` (replaces `SetFinalizer`)
- Swiss Tables map implementation
- `sync.Map` based on concurrent hash-trie
- `os.Root` / `os.OpenRoot` (directory-scoped FS)
- `testing.B.Loop()`, `T.Context()`, `B.Context()`, `T.Chdir()`, `B.Chdir()`
- `testing/synctest` (experimental)
- `slog.DiscardHandler`
- `encoding.TextAppender`, `encoding.BinaryAppender`
- `strings.Lines`, `SplitSeq`, `SplitAfterSeq`, `FieldsSeq`, `FieldsFuncSeq` (also in `bytes`)
- `crypto/sha3`, `crypto/hkdf`, `crypto/pbkdf2`, `crypto/mlkem`
- `crypto/rand.Text()`, `crypto/rand.Read` never fails
- `cipher.NewGCMWithRandomNonce`
- JSON `omitzero` struct tag
- `http.Server.Protocols`, `http.Transport.Protocols`
- Tool directives in `go.mod` (`go get -tool`)
- `go build/test -json` flag
- Build binary embeds VCS version info
- FIPS 140-3 support (`GODEBUG=fips140=on|only`)

### Go 1.25
- `testing/synctest.Test` (GA, replaces experimental `Run`)
- `encoding/json/v2` (experimental, `GOEXPERIMENT=jsonv2`)
- Container-aware GOMAXPROCS (respects cgroup CPU limits)
- Green Tea GC (experimental, `GOEXPERIMENT=greenteagc`)
- `http.CrossOriginProtection` (CSRF protection)
- `sync.WaitGroup.Go()`
- `trace.FlightRecorder`
- `os.Root` expanded methods (`Chmod`, `Chown`, `MkdirAll`, `RemoveAll`, `Rename`, `ReadFile`, `WriteFile`, `Symlink`, `Readlink`, `Link`)
- `reflect.TypeAssert[T]`
- `T.Attr()`, `T.Output()`, `B.Attr()`, `B.Output()`
- `slog.GroupAttrs`
- `hash.Cloner` interface
- `io/fs.ReadLinkFS`
- `runtime.SetDefaultGOMAXPROCS()`

### Go 1.26
- `new(expr)` -- `new` accepts expressions, not just types
- Recursive generic type constraints
- `errors.AsType[T]` -- generic, type-safe error checking
- Green Tea GC enabled by default
- Faster cgo (~30%), faster small allocations (~30%), optimized `io.ReadAll` (~2x)
- `fmt.Errorf` performance matches `errors.New`
- `crypto/hpke` (Hybrid Public Key Encryption, RFC 9180)
- Reader-less crypto (rand parameter ignored; use `nil`)
- `testing/cryptotest.SetGlobalRandom` for deterministic testing
- `simd/archsimd` (experimental, `GOEXPERIMENT=simd`, amd64 only)
- `runtime/secret` (experimental, `GOEXPERIMENT=runtimesecret`)
- Goroutine leak profile (experimental, `GOEXPERIMENT=goroutineleakprofile`)
- `bytes.Buffer.Peek(n)`
- `reflect.Type.Fields()`, `Methods()`, `Ins()`, `Outs()` iterators
- `reflect.Value.Fields()`, `Methods()` iterators
- `slog.NewMultiHandler`
- `T.ArtifactDir()`, `B.ArtifactDir()`
- `net.Dialer.DialTCP/UDP/IP/Unix` with context
- `netip.Prefix.Compare`
- `os.Process.WithHandle`
- `signal.NotifyContext` cause includes signal info
- `httptest.Server.Client` redirects `example.com` to test server
- `testing/cryptotest.SetGlobalRandom`
- `go fix` overhauled with modernization analyzers
- `cmd/doc` removed (use `go doc`)

---

## JSON Patterns

### Current Stable (encoding/json v1)

```go
// Use omitzero for struct fields (Go 1.24+)
type Event struct {
    Name    string    `json:"name"`
    Start   time.Time `json:"start,omitzero"`   // omits zero time.Time
    Count   int       `json:"count,omitempty"`   // omits 0
    Details *Info     `json:"details,omitzero"`  // omits nil pointer
}
```

`omitzero` checks `IsZero()` if available, otherwise checks for the zero value of the type.
`omitempty` uses legacy emptiness rules (0, "", nil, false, empty slice/map).
Prefer `omitzero` for struct types, especially `time.Time`.

### Experimental v2 (`GOEXPERIMENT=jsonv2`, Go 1.25+)

Key behavioral differences from v1:
- Nil slices marshal to `[]` (not `null`); use `FormatNilSliceAsNull(true)` for v1 behavior
- Nil maps marshal to `{}` (not `null`); use `FormatNilMapAsNull(true)` for v1 behavior
- Case-sensitive field matching by default; use `MatchCaseInsensitiveNames(true)` for v1 behavior
- Duplicate fields rejected by default; use `AllowDuplicateNames(true)` for v1 behavior
- Byte arrays marshal to base64 (not number arrays)
- Unmarshaling is 2.7x -- 10.2x faster than v1

---

## HTTP Routing Patterns (Go 1.22+)

```go
mux := http.NewServeMux()

mux.HandleFunc("GET /users", listUsers)           // method matching
mux.HandleFunc("POST /users", createUser)          // GET also registers HEAD
mux.HandleFunc("GET /users/{id}", getUser)         // path parameter
mux.HandleFunc("DELETE /users/{id}", deleteUser)
mux.HandleFunc("GET /files/{path...}", serveFile)  // wildcard rest
mux.HandleFunc("GET /exact/{$}", exactOnly)        // exact match (no prefix)

func getUser(w http.ResponseWriter, r *http.Request) {
    id := r.PathValue("id")  // extract path parameter
    // ...
}
```

Precedence: more specific patterns win. Method patterns beat non-method patterns.
Patterns with `{` and `}` are interpreted as wildcards; escape if needed.

---

## Timer/Ticker Patterns (Go 1.23+)

```go
// Safe: unreferenced timers are GC'd (Go 1.23+)
select {
case <-ch:
    // handle
case <-time.After(timeout):
    // timeout
}

// Reuse timer safely -- no channel drain needed (Go 1.23+)
t := time.NewTimer(d)
// ... later ...
t.Stop()
t.Reset(newDuration)
<-t.C
```

---

## Key GODEBUG Settings

| Setting | Default | Controls | Removal |
|---|---|---|---|
| `httpmuxgo121` | `0` | Revert to Go 1.21 ServeMux routing | -- |
| `asynctimerchan` | `0` | Timer channel buffering (0=unbuffered) | Go 1.27 |
| `randseednop` | `1` | `math/rand.Seed` is a no-op | -- |
| `fips140` | `off` | FIPS 140-3 mode (`off`/`on`/`only`) | -- |
| `cryptocustomrand` | `0` | Honor custom rand in crypto funcs | Future |
| `httpcookiemaxnum` | `3000` | Max cookies per HTTP request | -- |
| `containermaxprocs` | `1` | Cgroup CPU limits for GOMAXPROCS | -- |
| `updatemaxprocs` | `1` | Auto-update GOMAXPROCS from cgroup | -- |

---

## Performance Notes

- **Maps**: Swiss Tables since Go 1.24 (~30% faster for large maps, ~35% for pre-sized).
- **sync.Map**: Hash-trie since Go 1.24 (~49% faster geomean, especially modifications).
- **GC**: Green Tea GC default in Go 1.26 (10-40% less GC overhead).
- **Allocations**: Size-specialized malloc in Go 1.26 (~30% faster small allocs).
- **cgo**: ~30% less overhead in Go 1.26.
- **io.ReadAll**: ~2x faster, ~50% less memory in Go 1.26.
- **fmt.Errorf**: Now matches `errors.New` for non-formatted strings in Go 1.26.

These are automatic -- no code changes needed.
