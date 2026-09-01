# Modern Go Patterns -- Detailed Examples

Worked examples for Go 1.22 -- 1.27. The quick reference lives in [SKILL.md](../SKILL.md);
this file only goes deeper where the one-liner there is not enough.

- [Generics](#generics)
- [Iteration](#iteration)
- [Testing](#testing)
- [HTTP](#http)
- [JSON](#json)
- [Concurrency](#concurrency)
- [Cryptography](#cryptography)
- [Filesystem](#filesystem)
- [Logging](#logging)
- [Data types and utilities](#data-types-and-utilities)
- [Reflection](#reflection)
- [Modules and tooling](#modules-and-tooling)

---

## Generics

### Generic methods (1.27+)

Move a transform onto the type it belongs to instead of exporting a package-level helper:

```go
type Result[T any] struct {
    val T
    err error
}

// Before 1.27: func MapResult[T, U any](r Result[T], f func(T) U) Result[U]
func (r Result[T]) Map[U any](f func(T) U) Result[U] {
    if r.err != nil {
        return Result[U]{err: r.err}
    }
    return Result[U]{val: f(r.val)}
}
```

Constraints to keep in mind:

- Interface methods may not declare type parameters, and a generic method cannot implement
  an interface method. If a type must satisfy an interface, that method stays non-generic.
- A generic method must be instantiated before it is used as a value, exactly like a
  generic function.
- Receiver type parameters and method type parameters share one namespace; all non-blank
  names must be unique across the receiver and the signature.

### Promoted fields in struct literals (1.27+)

```go
type Audit struct {
    CreatedBy string
    CreatedAt time.Time
}
type Order struct {
    Audit
    ID    string
    Total int
}

o := Order{ID: "A1", Total: 500, CreatedBy: "svc"} // no nested Audit{...}
```

Rules: embedded types traversed to reach the field must not be pointer types, and you may
not set a promoted field if another key already sets the embedded struct it lives in
(`Order{Audit: a, CreatedBy: "svc"}` is invalid). `go fix -embedlit` performs the rewrite.

### Function type inference (1.27+)

Inference now runs wherever a generic function meets a concrete function type -- arguments,
conversions, return statements, assignments:

```go
sortFns := map[string]func(a, b int) int{}
sortFns["asc"] = cmp.Compare  // assignment: fine

func register(f func(a, b int) int) { ... }
register(cmp.Compare)         // argument: fine
```

In Go 1.27.0 the same inference inside a *composite literal element* crashes the compiler;
write `cmp.Compare[int]` there. See [SKILL.md](../SKILL.md#function-type-inference-everywhere).

---

## Iteration

### Defining iterators (1.23+)

```go
func Fibonacci(max int) iter.Seq[int] {
    return func(yield func(int) bool) {
        for a, b := 0, 1; a < max; a, b = b, a+b {
            if !yield(a) {
                return // consumer broke out; release resources here
            }
        }
    }
}
```

Always honour a `false` from `yield`: that is how `break`, `return` and errors propagate.
Return `iter.Seq2[K, V]` when callers need a key, index or error alongside each value.

### Pull iterators (1.23+)

Use when you need to interleave two sequences or read on demand:

```go
next, stop := iter.Pull(Fibonacci(100))
defer stop() // required, even on the happy path

for {
    v, ok := next()
    if !ok {
        break
    }
    use(v)
}
```

### Standard library iterators

```go
// slices (1.23+)
slices.All(s)        // iter.Seq2[int, V]
slices.Values(s)     // iter.Seq[V]
slices.Backward(s)   // reverse, index + value
slices.Chunk(s, n)   // iter.Seq[[]V]
slices.Collect(seq)  // seq -> slice
slices.Sorted(seq)   // seq -> sorted slice
slices.AppendSeq(dst, seq)

// maps (1.23+)
maps.All(m), maps.Keys(m), maps.Values(m)
maps.Collect(seq2)          // seq2 -> map
maps.Insert(dst, maps.All(src))

// strings / bytes (1.24+)
strings.Lines(s)            // including the trailing newline of each line
strings.SplitSeq(s, sep)
strings.SplitAfterSeq(s, sep)
strings.FieldsSeq(s)
strings.FieldsFuncSeq(s, f)

// sync.Map (1.23+): Range is itself a valid range expression
var m sync.Map
for k, v := range m.Range {
    use(k, v)
}
```

`go fix -stditerators` migrates `Len()`/`At(i)` loops over stdlib types to their `All()`
iterators automatically.

---

## Testing

### Benchmarks (1.24+)

```go
func BenchmarkProcess(b *testing.B) {
    data := expensiveSetup() // runs once
    for b.Loop() {           // only the loop body is timed
        process(data)        // result is kept alive; no sink variable needed
    }
}
```

`b.Loop()` removes the three classic mistakes at once: setup counted in the measurement,
a forgotten `b.ResetTimer`, and dead-code elimination of the thing under test.

### synctest (1.25+, `Sleep` 1.27+)

A bubble runs on a fake clock starting at midnight UTC 2000-01-01. Time advances only when
every goroutine in the bubble is durably blocked, so a `time.Hour` timeout resolves
instantly and deterministically.

```go
func TestTimeout(t *testing.T) {
    synctest.Test(t, func(t *testing.T) {
        ch := make(chan int)
        _, err := ReadWithTimeout(ch, time.Minute) // returns immediately
        if err == nil {
            t.Fatal("expected timeout")
        }
    })
}

func TestSettles(t *testing.T) {
    synctest.Test(t, func(t *testing.T) {
        go worker()

        synctest.Sleep(time.Second) // 1.27: time.Sleep + synctest.Wait
        // Every other goroutine has now blocked; assertions are race-free.

        synctest.Wait() // 1.25: wait without advancing the clock
    })
}
```

Inside a bubble: no `t.Run`, `t.Parallel` or `t.Deadline`. `t.Cleanup` runs inside the
bubble, and `t.Context()` is cancelled with it.

Prefer `synctest.Sleep` over bare `time.Sleep`: when the test and the system under test
sleep for the same duration, plain `Sleep` leaves the winner unspecified.

### In-memory HTTP servers (1.27+)

`httptest.NewTestServer` uses a fake in-memory network, so it works inside a synctest
bubble -- something `httptest.NewServer` cannot do -- and registers its own cleanup.

```go
func TestClientRetry(t *testing.T) {
    synctest.Test(t, func(t *testing.T) {
        srv := httptest.NewTestServer(t, http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            time.Sleep(2 * time.Second) // fake clock: free
            io.WriteString(w, "ok")
        }))
        // No defer srv.Close(). Any host works; srv.URL is "http://example.com".
        resp, err := srv.Client().Get("http://www.example.com/")
        ...
    })
}
```

A handler panic fails the test (except `http.ErrAbortHandler`). Configure via `srv.Config`
before the first call to `Client`, `Start` or `StartTLS`.

### Test metadata and artifacts

```go
func TestFeature(t *testing.T) {
    t.Chdir(t.TempDir())       // 1.24: restored afterwards
    t.Attr("team", "platform") // 1.25: surfaces in go test -json
    t.Attr("issue", "PROJ-1234")

    logger := slog.New(slog.NewTextHandler(t.Output(), nil)) // 1.25

    dir := t.ArtifactDir()     // 1.26: go test -artifacts -outputdir=...
    os.WriteFile(filepath.Join(dir, "debug.log"), data, 0o644)
}
```

### Deterministic crypto in tests (1.26+)

```go
func TestSigning(t *testing.T) {
    cryptotest.SetGlobalRandom(t, 42) // same keys every run
    key, _ := ecdsa.GenerateKey(elliptic.P256(), nil)
    ...
}
```

This replaces threading a fake `io.Reader` through the code and, in 1.27, the deprecated
`tls.Config.Rand`.

---

## HTTP

### Routing (1.22+)

Patterns are in [SKILL.md](../SKILL.md#http-routing-122). Two rules that catch people out:
precedence is by specificity rather than registration order (and a method-qualified pattern
beats an unqualified one), and `{`/`}` are wildcard syntax, so a literal brace in a path
must be escaped.

### Protocol configuration (1.24+)

```go
srv := &http.Server{Handler: mux}
srv.Protocols = new(http.Protocols)
srv.Protocols.SetHTTP1(true)
srv.Protocols.SetHTTP2(true)

tr := http.DefaultTransport.(*http.Transport).Clone()
tr.Protocols = new(http.Protocols)
tr.Protocols.SetHTTP1(true)
tr.Protocols.SetHTTP2(true)
```

### Server hardening (1.26+ / 1.27+)

```go
srv := &http.Server{
    Handler:             mux,
    MaxHeaderValueCount: 100, // 1.27; default DefaultMaxHeaderValueCount = 500
}
```

`net/http` also caps cookies per request (3000, `GODEBUG=httpcookiemaxnum`) and `net/url`
caps query parameters (10000, `GODEBUG=urlmaxqueryparams`) since 1.26.

HTTP/2 servers honour RFC 9218 client priority signals from 1.27; set
`Server.DisableClientPriority = true` for the old round-robin scheduling.

### Response bodies (1.27+)

```go
resp, err := client.Do(req)
if err != nil {
    return err
}
defer resp.Body.Close() // drains up to 256 KiB / 50 ms so the connection is reused
```

Delete manual `io.Copy(io.Discard, resp.Body)` drains. `Close` is still mandatory.

### Reverse proxy (1.26+)

```go
proxy := &httputil.ReverseProxy{
    Rewrite: func(r *httputil.ProxyRequest) { // Director is deprecated
        r.SetURL(backend)
        r.SetXForwarded()  // sets X-Forwarded-For/Host/Proto safely
    },
}
```

`Rewrite` exists because `Director` could not see the inbound request, which made
`X-Forwarded-*` handling easy to get wrong.

### Cookies (1.23+)

```go
cookies, err := http.ParseCookie("session=abc; theme=dark")
c, err := http.ParseSetCookie("session=abc; Secure; Partitioned; Path=/")
named := r.CookiesNamed("session")
```

---

## JSON

### omitzero vs omitempty (1.24+)

```go
type Event struct {
    Name  string     `json:"name"`
    Start time.Time  `json:"start,omitzero"` // omitempty never omits a zero time.Time
    End   *time.Time `json:"end,omitzero"`
    Count int        `json:"count,omitempty"`
}

type Status struct{ Code int }

func (s Status) IsZero() bool { return s.Code == 0 }

type Response struct {
    Status Status `json:"status,omitzero"` // uses IsZero
}
```

`omitzero` is defined by the Go zero value (or `IsZero`); `omitempty` by JSON emptiness.
They differ for bools, numbers, pointers and structs -- migrate those to `omitzero`.

### encoding/json/v2 (1.27+)

v2 is generally available and v1 is now implemented on top of it. v1 is not deprecated:
keep it where an existing wire format depends on its quirks.

```go
import (
    "encoding/json/jsontext"
    json "encoding/json/v2"
)

// Streaming, without an intermediate []byte.
err := json.UnmarshalRead(r, &v, json.RejectUnknownMembers(true))
err = json.MarshalWrite(w, v, jsontext.WithIndent("  "))

// Encoder/Decoder level, for framing many values.
enc := jsontext.NewEncoder(w)
err = json.MarshalEncode(enc, v)

dec := jsontext.NewDecoder(r)
err = json.UnmarshalDecode(dec, &v)
```

Tags: `omitzero`, `omitempty`, `string`, `case:ignore` / `case:strict`, and `embed`
(flatten a nested struct; renamed from `inline` before release). The `format` and `unknown`
tag options were dropped during development -- do not use them.

Custom behaviour without defining a type:

```go
yesNo := json.MarshalToFunc(func(enc *jsontext.Encoder, b bool) error {
    if b {
        return enc.WriteToken(jsontext.String("yes"))
    }
    return enc.WriteToken(jsontext.String("no"))
})
out, err := json.Marshal(v, json.WithMarshalers(yesNo))
```

### v1 -> v2 behaviour changes

Every difference has an option that restores v1; `json.DefaultOptionsV1()` restores all of
them at once.

| Behaviour | v1 | v2 | Restore v1 with |
|---|---|---|---|
| Field name matching | case-insensitive | case-sensitive | `MatchCaseInsensitiveNames(true)` or `case:ignore` |
| Duplicate object names | accepted | error | `jsontext.AllowDuplicateNames(true)` |
| Invalid UTF-8 | replaced silently | error | `jsontext.AllowInvalidUTF8(true)` |
| `nil` slice / map | `null` | `[]` / `{}` | `FormatNilSliceAsNull(true)`, `FormatNilMapAsNull(true)` |
| `omitempty` | Go emptiness | JSON emptiness | `OmitEmptyWithLegacySemantics(true)` |
| `string` tag | any scalar | numbers only | `StringifyWithLegacySemantics(true)` |
| Byte array `[N]byte` | array of numbers | base64 string | `FormatByteArrayAsArray(true)` |
| Go array length | any JSON length | must match | `UnmarshalArrayFromAnyLength(true)` |
| Map ordering | deterministic | unspecified | `Deterministic(true)` |
| HTML/JS escaping | always | only when required | `jsontext.EscapeForHTML(true)` |
| `time.Duration` | nanosecond number | error | `FormatDurationAsNano(true)` |
| Unmarshal `null` into a value | inconsistent | always zeroes | `MergeWithLegacySemantics(true)` |
| `MarshalJSON` on pointer receiver | needs addressability | always callable | `CallMethodsWithLegacySemantics(true)` |
| Malformed struct tags | ignored | runtime error | `ReportErrorsWithLegacySemantics(true)` |

`GOEXPERIMENT=nojsonv2` reverts the whole implementation if a v1 caller regresses; it is
expected to be removed in a later release, so file an issue rather than settling there.

---

## Concurrency

### WaitGroup.Go (1.25+)

```go
var wg sync.WaitGroup
wg.Go(func() { a = fetchAPI() })
wg.Go(func() { b = queryDB() })
wg.Wait()
```

Use `golang.org/x/sync/errgroup` when you need error propagation or a concurrency limit;
`wg.Go` covers the fire-and-join case.

### Atomic types (1.19+)

```go
var hits atomic.Int64 // not: var hits int64 + atomic.AddInt64(&hits, 1)
hits.Add(1)
n := hits.Load()
```

The typed wrappers forbid accidental non-atomic access and fix 64-bit alignment on 32-bit
platforms. `go fix -atomictypes` performs the conversion.

### unique (1.23+)

```go
type Request struct {
    Method unique.Handle[string]
    Host   unique.Handle[string]
}

r := Request{Method: unique.Make(m), Host: unique.Make(h)}
r1.Host == r2.Host // pointer comparison, no string compare
```

### weak pointers + AddCleanup (1.24+)

```go
type Cache[K comparable, V any] struct {
    mu    sync.Mutex
    items map[K]weak.Pointer[V]
}

func (c *Cache[K, V]) Set(key K, val *V) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.items[key] = weak.Make(val)
    runtime.AddCleanup(val, func(k K) { // never resurrects val
        c.mu.Lock()
        defer c.mu.Unlock()
        delete(c.items, k)
    }, key)
}
// Get looks up c.items[key] and returns wp.Value(), which is nil once collected.
```

`AddCleanup` beats `SetFinalizer` on every axis: many cleanups per object, no resurrection,
works with interior pointers, and objects in a cycle stay collectable.

### Timers (1.23+)

```go
t := time.NewTimer(d)
t.Stop()
t.Reset(next) // no channel drain -- channels are unbuffered since 1.23
<-t.C

for {
    select {
    case v := <-ch:
        use(v)
    case <-time.After(timeout): // GC-eligible; safe in a loop
        return errTimeout
    case <-ctx.Done():
        return ctx.Err()
    }
}
```

The `asynctimerchan` escape hatch was removed in 1.27; timer channels are always
synchronous now.

### Finding leaks (1.27+)

```
go tool pprof http://localhost:6060/debug/pprof/goroutineleak
```

Reports goroutines blocked on a channel, mutex or condition variable that no runnable
goroutine can reach. It cannot see leaks whose primitive is reachable from a global.

---

## Cryptography

### Tokens and derivation (1.24+)

```go
token := rand.Text() // crypto/rand: base32, >=128 bits, never fails

// Both take the hash constructor first and return (key, error).
key, err := hkdf.Key(sha256.New, secret, salt, info, 32)   // secret, salt []byte; info string
dk, err := pbkdf2.Key(sha256.New, password, salt, 600_000, 32) // password string
sum := sha3.Sum256(data)

shake := sha3.NewSHAKE256() // NewSHAKE, not NewShake
shake.Write(data)
shake.Read(out)
```

### Reader-less crypto (1.26+)

Randomness parameters are ignored; pass `nil` so the code reads honestly.

```go
ec, _ := ecdsa.GenerateKey(elliptic.P256(), nil)
rsaKey, _ := rsa.GenerateKey(nil, 2048)
x, _ := ecdh.P256().GenerateKey(nil)
p, _ := rand.Prime(nil, 64)
```

`ed25519.GenerateKey(rand)` still honours a non-nil reader.

### AEAD (1.24+)

```go
block, _ := aes.NewCipher(key)
aead, _ := cipher.NewGCMWithRandomNonce(block)
ct := aead.Seal(nil, nil, plaintext, aad) // nonce generated and prepended
pt, err := aead.Open(nil, nil, ct, aad)
```

This removes the single most common AES-GCM bug: a reused or predictable nonce.

### HPKE (1.26+)

```go
kdf, aead := hpke.HKDFSHA256(), hpke.AES256GCM()
ct, err := hpke.Seal(pub, kdf, aead, info, plaintext)
pt, err := hpke.Open(priv, kdf, aead, info, ct)
```

### Post-quantum signatures (1.27+)

```go
sk, err := mldsa.GenerateKey(mldsa.MLDSA65()) // FIPS 204; also MLDSA44 / MLDSA87
sig, err := sk.Sign(nil, msg, &mldsa.Options{})
err = mldsa.Verify(sk.PublicKey(), msg, sig, &mldsa.Options{})
```

`sk.Bytes()` is a 32-byte seed; `NewPrivateKey(params, seed)` restores it. `crypto/x509`
parses ML-DSA keys and certificates, and TLS 1.3 negotiates `MLDSA44/65/87`.

Key exchange is separate: X25519MLKEM768 has been on by default since 1.24, the SecP hybrids
since 1.26, and `MLKEM1024` is opt-in through `Config.CurvePreferences` since 1.27.

---

## Filesystem

### os.Root (1.24+, expanded 1.25+)

```go
root, err := os.OpenRoot(baseDir)
if err != nil {
    return err
}
defer root.Close()

data, err := root.ReadFile(userSuppliedName) // cannot escape baseDir
root.WriteFile("out.json", data, 0o644)
root.MkdirAll("a/b/c", 0o755)   // 1.25+
root.RemoveAll("tmp")           // 1.25+
root.Rename("old", "new")       // 1.25+
root.Symlink("target", "link")  // 1.25+
fsys := root.FS()               // fs.FS view
```

Symlinks, `..` and absolute paths that leave the root all fail. This is the correct answer
to path traversal -- `filepath.Clean` plus a prefix check is not.

Requires a patched toolchain: escapes were fixed in 1.24.3 (CVE-2025-22873) and again in
1.25.12 / 1.26.5 (CVE-2026-39822).

### os.CopyFS (1.23+)

```go
//go:embed templates
var templates embed.FS

os.CopyFS("/var/templates", templates)
os.CopyFS(dst, os.DirFS(src))
```

---

## Logging

Handlers and `GroupAttrs` are in [SKILL.md](../SKILL.md#structured-logging-124--126).
`GroupAttrs` matters because `slog.Group` takes `...any`, so building a group from a
`[]slog.Attr` used to mean an allocation and a loop.

From 1.27, tracebacks of `go 1.27` modules include `runtime/pprof` goroutine labels in the
header line (`GODEBUG=tracebacklabels=0` opts out) -- label long-lived goroutines and crash
dumps get much easier to read.

---

## Data types and utilities

### uuid (1.27+)

`New` is v4 (122 random bits); `NewV7` prefixes a 48-bit timestamp, so values sort by
creation time -- prefer it for database keys, where random v4s scatter index inserts.
`Parse` also accepts the braced, `urn:uuid:` and unhyphenated forms.

`UUID` is `[16]byte`: comparable, usable as a map key, and a `TextMarshaler`/`TextAppender`.
`Compare` sorts big-endian per RFC 9562, so `slices.SortFunc(ids, uuid.UUID.Compare)` works
directly.

### CutLast (1.27+)

```go
if dir, file, ok := strings.CutLast(path, "/"); ok { // also bytes.CutLast
    use(dir, file)
}
// Not found: returns (path, "", false).
```

### url.Clone (1.27+)

```go
next := u.Clone() // deep copy, including the parsed query
next.Path = "/v2" + u.Path
q := u.Query().Clone()
```

### maphash.Hasher (1.27+)

Lets non-comparable types, or a custom equivalence relation, back a hash-based container:

```go
type CaseInsensitive struct{}

func (CaseInsensitive) Hash(h *maphash.Hash, s string) { h.WriteString(strings.ToLower(s)) }
func (CaseInsensitive) Equal(x, y string) bool         { return strings.ToLower(x) == strings.ToLower(y) }

var _ maphash.Hasher[string] = CaseInsensitive{}
var _ maphash.Hasher[int] = maphash.ComparableHasher[int]{} // == and the default hash
```

`Equal(x, y)` implies equal hashes, and a `Hasher` must be stateless. The per-container
seed is what makes hash-flooding attacks impractical.

### big.Int.Divide (1.27+)

```go
q, r := new(big.Int).Divide(x, y, new(big.Int), big.Floor) // or Trunc, Round, Ceil
```

One call replaces the `Quo`/`Rem` versus `Div`/`Mod` sign-handling trap.

---

## Reflection

```go
typ := reflect.TypeFor[MyStruct]() // 1.22: not reflect.TypeOf(MyStruct{})

for f := range typ.Fields() {      // 1.26 iterators
    use(f.Name, f.Type)
}
for m := range typ.Methods() { ... }
for p := range reflect.TypeFor[func(int) error]().Ins() { ... }

for f, v := range reflect.ValueOf(x).Fields() { // field metadata + value
    use(f.Name, v.Interface())
}

if p, ok := reflect.TypeAssert[Person](val); ok { ... } // 1.25: no Interface().(T)
```

---

## Modules and tooling

Commands (`go fix`, `go get -tool`, `go doc`, `stdversion`) are covered in
[SKILL.md](../SKILL.md#toolchain). What belongs here is the module file itself:

```
go 1.27              // language and stdlib floor; raise it deliberately
toolchain go1.27.0   // the toolchain to fetch if the local one is older
godebug default=go1.27
godebug tlsmlkem=0   // per-setting overrides, scoped to this module
```

`go mod tidy` collapses `require` blocks to two (direct and indirect) for `go 1.27`
modules, preserving attached comments. From 1.27 the `go` command also accepts a removed
GODEBUG in `go.mod` or a `//go:debug` comment as long as it is set to the value that was
its default when it was removed, so modules that pinned a supported setting keep building.

`//go:fix inline` on a deprecated function makes `go fix` rewrite call sites in downstream
modules -- the supported way to retire an API without breaking callers:

```go
// Deprecated: use NewClient.
//
//go:fix inline
func New() *Client { return NewClient(defaultOpts) }
```
