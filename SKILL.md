---
name: go-latest-version
description: >
  Enforces modern Go standards and idioms for Go 1.22 through 1.27. Use when writing,
  reviewing, or modifying Go code to ensure the latest language features, standard library
  APIs, and best practices are used. Prevents deprecated patterns and guides toward
  idiomatic modern Go.
license: Apache-2.0
metadata:
  author: FumingPower3925
  version: "2.0"
  go-versions: "1.22, 1.23, 1.24, 1.25, 1.26, 1.27"
---

# Modern Go Standards (1.22 -- 1.27)

Two rules govern everything below.

1. **The `go` line in `go.mod` is the floor.** Anything tagged `1.27` in the tables fails to
   compile under `go 1.26`. Read `go.mod` before reaching for a version-gated feature. With no
   `go.mod`, assume Go 1.27.
2. **A replacement in these tables is not optional.** The old form is what `go fix`, `go vet`
   and reviewers flag -- not a matter of taste.

Deeper examples: [MODERN-PATTERNS.md](references/MODERN-PATTERNS.md).
Removals, GODEBUG, platforms: [DEPRECATED.md](references/DEPRECATED.md).

---

## Non-negotiables

1. Never use a deprecated API when a modern replacement exists.
2. `for i := range n`, never `for i := 0; i < n; i++`. (1.22)
3. Delete every `v := v` loop-capture hack; loop vars are per-iteration. (1.22)
4. `math/rand/v2`, never `math/rand`. (1.22)
5. Model sequences as `iter.Seq` / `iter.Seq2`, not callback APIs. (1.23)
6. Any path derived from untrusted input goes through `os.Root`. (1.24)
7. `for b.Loop()`, never `for range b.N`. (1.24)
8. `t.Context()` / `t.Chdir()` in tests; never hand-roll either. (1.24)
9. `omitzero`, not `omitempty`, on struct and `time.Time` fields. (1.24)
10. `runtime.AddCleanup`, never `runtime.SetFinalizer`. (1.24)
11. `crypto/rand.Text()` for tokens; never a hand-written generator. (1.24)
12. `wg.Go(f)`, never `wg.Add(1)` + `defer wg.Done()`. (1.25)
13. Concurrency tests use `testing/synctest`; never `time.Sleep` to "let it settle". (1.25)
14. `errors.AsType[T](err)`, never `errors.As(err, &target)`. (1.26)
15. `new(expr)` for inline pointers; no one-shot temporaries. (1.26)
16. Pass `nil` for every crypto `rand io.Reader` parameter. (1.26)
17. Stdlib `uuid`, never `github.com/google/uuid` or friends. (1.27)
18. `encoding/json/v2` for new code; `encoding/json` only for existing wire formats. (1.27)
19. Run `go fix ./...` before calling a change done. (1.26)
20. Target Go 1.27; only 1.26 and 1.27 still receive security fixes.

---

## Replace old with new

### Language and syntax

| Instead of (old) | Use (modern) | Since |
|---|---|---|
| `for i := 0; i < n; i++` | `for i := range n` | 1.22 |
| `v := v` inside a loop body | Delete it | 1.22 |
| `interface{}` | `any` | 1.18 |
| `if a > b { m = a } else { m = b }` | `max(a, b)` / `min(a, b)` | 1.21 |
| `p := 42; foo(&p)` | `foo(new(42))` | 1.26 |
| Package-level generic helper tied to one type | Generic **method** on that type | 1.27 |
| `T{Embedded: E{Field: v}}` | `T{Field: v}` (promoted field key) | 1.27 |
| `cmp.Compare[int]` in a func-typed position | `cmp.Compare` (inferred) | 1.27 |

### Iteration

| Instead of (old) | Use (modern) | Since |
|---|---|---|
| Callback-style `Each(func(v T) bool)` | `iter.Seq[V]` / `iter.Seq2[K, V]` | 1.23 |
| `x.Len()` / `x.At(i)` index loops | The type's `All()` iterator | 1.23 |
| `for i := len(s)-1; i >= 0; i--` | `for i, v := range slices.Backward(s)` | 1.23 |
| Manual `for k, v := range src { dst[k] = v }` | `maps.Copy` / `Clone` / `Insert` / `Collect` | 1.23 |
| `sort.Slice(s, less)` | `slices.SortFunc(s, cmp)` / `slices.Sort(s)` | 1.21 |
| Manual "does it contain" loop | `slices.Contains` / `ContainsFunc` | 1.21 |
| `for _, x := range strings.Split(s, ",")` | `for x := range strings.SplitSeq(s, ",")` | 1.24 |
| `for _, w := range strings.Fields(s)` | `for w := range strings.FieldsSeq(s)` | 1.24 |
| `bufio.Scanner` just to split a string | `for line := range strings.Lines(s)` | 1.24 |

### Strings and bytes

| Instead of (old) | Use (modern) | Since |
|---|---|---|
| `HasPrefix` + `TrimPrefix` | `strings.CutPrefix` (and `CutSuffix`) | 1.20 |
| `Index` + manual slicing | `strings.Cut` | 1.18 |
| `LastIndex` + manual slicing | `strings.CutLast` / `bytes.CutLast` | 1.27 |
| `s += x` in a loop | `strings.Builder` | 1.10 |
| `fmt.Sprintf("%s:%d", host, port)` for dialing | `net.JoinHostPort(host, port)` -- IPv6-safe | -- |

### Errors

| Instead of (old) | Use (modern) | Since |
|---|---|---|
| `var t *T; errors.As(err, &t)` | `t, ok := errors.AsType[*T](err)` | 1.26 |
| `github.com/pkg/errors` wrapping | `fmt.Errorf("...: %w", err)` | 1.13 |
| `go-multierror` / manual joins | `errors.Join(errs...)` | 1.20 |
| Avoiding `fmt.Errorf` on perf grounds | Use it; it now matches `errors.New` | 1.26 |

### Concurrency and runtime

| Instead of (old) | Use (modern) | Since |
|---|---|---|
| `wg.Add(1); go func(){ defer wg.Done(); f() }()` | `wg.Go(f)` | 1.25 |
| `var n int64; atomic.AddInt64(&n, 1)` | `var n atomic.Int64; n.Add(1)` | 1.19 |
| `runtime.SetFinalizer` | `runtime.AddCleanup` | 1.24 |
| Map-based string dedup | `unique.Make(s)` | 1.23 |
| Hand-rolled expiring cache | `weak.Pointer[T]` + `runtime.AddCleanup` | 1.24 |
| Drain a timer channel before `Reset` | `t.Stop(); t.Reset(d)` -- no drain | 1.23 |
| Avoiding `time.After` in loops | Safe; unreferenced timers are collected | 1.23 |
| Manual `GOMAXPROCS` from cgroup limits | Automatic; `runtime.SetDefaultGOMAXPROCS()` to override | 1.25 |
| `unsafe.Pointer(uintptr(p) + uintptr(n))` | `unsafe.Add(p, n)` | 1.17 |

### Testing

| Instead of (old) | Use (modern) | Since |
|---|---|---|
| `for range b.N` (+ `ResetTimer`, + sink var) | `for b.Loop()` | 1.24 |
| `context.WithCancel` in a test | `t.Context()` | 1.24 |
| Save/restore working directory | `t.Chdir(dir)` | 1.24 |
| `time.Sleep` to sequence goroutines | `synctest.Test` + `synctest.Wait` | 1.25 |
| `time.Sleep` inside a bubble | `synctest.Sleep(d)` (sleep **and** wait) | 1.27 |
| `synctest.Run` | `synctest.Test` | 1.25 |
| `httptest.NewServer` in a synctest bubble | `httptest.NewTestServer(t, h)` (in-memory net) | 1.27 |
| `defer srv.Close()` on a test server | Nothing; `NewTestServer` registers cleanup | 1.27 |
| Seeding crypto by hand for determinism | `cryptotest.SetGlobalRandom(t, seed)` | 1.26 |
| Writing debug output to `t.TempDir()` | `t.ArtifactDir()` | 1.26 |

### HTTP and net

| Instead of (old) | Use (modern) | Since |
|---|---|---|
| Third-party router for methods/params | `mux.HandleFunc("GET /items/{id}", h)` + `r.PathValue` | 1.22 |
| Custom CSRF middleware | `http.NewCrossOriginProtection()` | 1.25 |
| `io.Copy(io.Discard, resp.Body)` before `Close` | Just `Close`; it drains (<=256 KiB, <=50 ms) | 1.27 |
| `httputil.ReverseProxy.Director` | `ReverseProxy.Rewrite` | 1.26 |
| Manual HTTP/2 wiring | `Server.Protocols` / `Transport.Protocols` | 1.24 |
| Hand-parsing `Cookie` headers | `http.ParseCookie` / `ParseSetCookie` | 1.23 |
| Unbounded request header parsing | `Server.MaxHeaderValueCount` (default 500) | 1.27 |

### JSON

| Instead of (old) | Use (modern) | Since |
|---|---|---|
| `omitempty` on a struct / `time.Time` / pointer | `omitzero` | 1.24 |
| `encoding/json` for a **new** format | `encoding/json/v2` | 1.27 |
| `json.Marshal` + `w.Write` | `json.MarshalWrite(w, v)` | 1.27 |
| `json.NewDecoder(r).Decode(&v)` | `json.UnmarshalRead(r, &v)` | 1.27 |
| `DisallowUnknownFields` on a Decoder | `json.RejectUnknownMembers(true)` option | 1.27 |
| Third-party fast JSON codecs | v2 (`Unmarshal` is much faster than v1) | 1.27 |

### Crypto

| Instead of (old) | Use (modern) | Since |
|---|---|---|
| Hand-rolled token generator | `crypto/rand.Text()` | 1.24 |
| `golang.org/x/crypto/{sha3,hkdf,pbkdf2}` | `crypto/{sha3,hkdf,pbkdf2}` | 1.24 |
| `cipher.NewGCM` + manual nonce plumbing | `cipher.NewGCMWithRandomNonce` | 1.24 |
| `ecdsa.GenerateKey(curve, rand.Reader)` | `ecdsa.GenerateKey(curve, nil)` | 1.26 |
| `rsa.GenerateKey(rand.Reader, bits)` | `rsa.GenerateKey(nil, bits)` | 1.26 |
| `rsa.EncryptPKCS1v15` | `rsa.EncryptOAEP` | 1.26 |
| Rolling your own hybrid encryption | `crypto/hpke` (RFC 9180) | 1.26 |
| `tls.Config.Rand` | `cryptotest.SetGlobalRandom` | 1.27 |
| Classical-only signatures for long-lived data | `crypto/mldsa` (FIPS 204) | 1.27 |

### Filesystem and reflection

| Instead of (old) | Use (modern) | Since |
|---|---|---|
| `filepath.Clean` + prefix checks | `os.OpenRoot(dir)` + `root.Open(name)` | 1.24 |
| Hand-written recursive copy | `os.CopyFS(dst, srcFS)` | 1.23 |
| `reflect.TypeOf(x)` with a static type | `reflect.TypeFor[T]()` | 1.22 |
| `for i := range t.NumField()` | `for f := range t.Fields()` | 1.26 |
| `v.Interface().(T)` | `reflect.TypeAssert[T](v)` | 1.25 |

### Logging

| Instead of (old) | Use (modern) | Since |
|---|---|---|
| `logrus` / `zap` in new code | `log/slog` | 1.21 |
| `slog.NewTextHandler(io.Discard, nil)` | `slog.DiscardHandler` | 1.24 |
| `slog.Group("k", anySlice...)` | `slog.GroupAttrs("k", attrs...)` | 1.25 |
| Custom tee handler | `slog.NewMultiHandler(h1, h2)` | 1.26 |

### Drop the dependency

| Third-party | Standard library | Since |
|---|---|---|
| `github.com/google/uuid`, `gofrs/uuid` | `uuid` | 1.27 |
| `golang.org/x/exp/{slices,maps}` | `slices`, `maps` | 1.21 / 1.23 |
| `golang.org/x/exp/constraints` | `cmp.Ordered` | 1.21 |
| `golang.org/x/exp/rand` | `math/rand/v2` | 1.22 |
| `golang.org/x/crypto/{sha3,hkdf,pbkdf2}` | `crypto/{sha3,hkdf,pbkdf2}` | 1.24 |
| `gorilla/mux` for method + path routing | `net/http.ServeMux` | 1.22 |
| A `tools.go` file with blank imports | `go get -tool` + `go tool` | 1.24 |

---

## Go 1.27 syntax

Requires `go 1.27` in `go.mod`.

### Generic methods

A method may now declare its own type parameters, so a transform that belongs to a type
stops leaking into the package namespace.

```go
type List[E any] []E

// Apply returns the list obtained from applying f to each element of l.
func (l List[E]) Apply[F any](f func(E) F) List[F] {
    r := make(List[F], len(l))
    for i, x := range l {
        r[i] = f(x)
    }
    return r
}

names := List[User]{u1, u2}.Apply(User.Name) // List[string]
```

Interface methods still cannot declare type parameters, and a generic method cannot
implement an interface method. Anything that must satisfy an interface stays non-generic.

### Promoted fields as struct-literal keys

```go
type Meta struct{ Name string }
type Record struct {
    Meta
    ID int
}

r := Record{ID: 7, Name: "x"} // was: Record{ID: 7, Meta: Meta{Name: "x"}}
```

Two limits: the embedded types traversed must not be pointers, and a key may not name a
promoted field of an embedded struct that another key sets wholesale.

### Function type inference everywhere

Assigning, converting, passing or returning a generic function now infers its type
arguments from the target function type.

```go
func sortBy(less func(a, b int) int) { ... }

func wire() func(a, b int) int {
    sortBy(cmp.Compare)                     // argument
    f := (func(a, b int) int)(cmp.Compare)  // conversion
    _ = f
    return cmp.Compare                      // return
}
```

Known Go 1.27.0 compiler bug: inference in a **composite-literal element**
(`S{less: cmp.Compare}`, a map or slice literal entry) crashes the compiler with
`internal compiler error: ... is not assignable`. Instantiate explicitly there:
`S{less: cmp.Compare[int]}`.

---

## Snippets

### Iterators (1.23+)

```go
func Reversed[V any](s []V) iter.Seq[V] {
    return func(yield func(V) bool) {
        for i := len(s) - 1; i >= 0; i-- {
            if !yield(s[i]) {
                return
            }
        }
    }
}

for v := range Reversed(s) { ... }

for i, v := range slices.All(s) { }      // index + value
for v := range slices.Values(s) { }      // values
for i, v := range slices.Backward(s) { } // reverse
for c := range slices.Chunk(s, 3) { }    // chunks
for k := range maps.Keys(m) { }          // keys
sorted := slices.Sorted(maps.Keys(m))    // collect + sort
```

### math/rand/v2 (1.22+, generic method 1.27+)

```go
n := rand.IntN(100)                  // [0, 100)
d := rand.N(100 * time.Millisecond)  // generic: any integer-like type
j := r.N(time.Second)                // 1.27: same, as a method on *rand.Rand
```

Default source is ChaCha8 and auto-seeded. Never call `Seed`.

### HTTP routing (1.22+)

```go
mux := http.NewServeMux()
mux.HandleFunc("GET /users", listUsers)          // GET also registers HEAD
mux.HandleFunc("POST /users", createUser)
mux.HandleFunc("GET /users/{id}", getUser)       // r.PathValue("id")
mux.HandleFunc("GET /files/{path...}", serveAll) // rest of the path
mux.HandleFunc("GET /health/{$}", health)        // exact, no prefix match
```

More specific patterns win; method patterns beat method-less ones.

### CSRF (1.25+)

```go
p := http.NewCrossOriginProtection()
p.AddTrustedOrigin("https://myapp.example.com")
http.ListenAndServe(":8080", p.Handler(mux))
// GET/HEAD/OPTIONS always pass through.
```

### UUIDs (1.27+)

```go
import "uuid"

id := uuid.New()   // v4, 122 random bits
key := uuid.NewV7() // timestamp-prefixed: sorts by creation time -- prefer for DB keys
u, err := uuid.Parse(s)
```

`UUID` is `[16]byte`: comparable, usable as a map key, and text-marshalable.

### JSON (1.24+ / 1.27+)

```go
type Event struct {
    Name  string    `json:"name"`
    Start time.Time `json:"start,omitzero"` // omitempty cannot omit a zero time.Time
    Count int       `json:"count,omitempty"`
}

// v2: stream, and reject junk instead of silently ignoring it.
err := json.UnmarshalRead(r, &e, json.RejectUnknownMembers(true))
err = json.MarshalWrite(w, e, jsontext.WithIndent("  "))
```

v2 defaults differ from v1 (nil slice -> `[]`, case-sensitive names, duplicate names and
invalid UTF-8 rejected). Do not switch an existing wire format without reading the
migration table in [MODERN-PATTERNS.md](references/MODERN-PATTERNS.md).

### Testing (1.24+ .. 1.27+)

```go
func BenchmarkProcess(b *testing.B) {
    data := expensiveSetup() // once, not b.N times
    for b.Loop() {           // no ResetTimer, no sink variable
        process(data)
    }
}

func TestTimeout(t *testing.T) {
    synctest.Test(t, func(t *testing.T) {
        srv := httptest.NewTestServer(t, handler) // in-memory net, auto-closed
        resp, err := srv.Client().Get("http://www.example.com/")
        // A time.Hour sleep in the handler resolves instantly.
        ...
    })
}
```

Inside a bubble: no `t.Run`, `t.Parallel` or `t.Deadline`; use `synctest.Sleep` over
`time.Sleep` so the system under test settles before you assert.

### Safe filesystem access (1.24+)

```go
root, err := os.OpenRoot("/var/data")
defer root.Close()

data, _ := root.ReadFile("config.json")
root.MkdirAll("a/b/c", 0o755)       // 1.25+
root.Rename("old", "new")           // 1.25+
_, err = root.Open("../etc/passwd") // error: escapes the root
```

### Crypto (1.24+ / 1.26+)

```go
token := rand.Text()                                // crypto/rand, base32, 128+ bits
ecKey, _ := ecdsa.GenerateKey(elliptic.P256(), nil) // nil rand (1.26+)
rsaKey, _ := rsa.GenerateKey(nil, 2048)

block, _ := aes.NewCipher(key)
aead, _ := cipher.NewGCMWithRandomNonce(block)      // nonce generated and prepended
ct := aead.Seal(nil, nil, plaintext, aad)

dk, _ := pbkdf2.Key(sha256.New, pw, salt, 600_000, 32) // hash first, then the password
```

### Structured logging (1.24+ .. 1.26+)

```go
logger := slog.New(slog.NewMultiHandler(         // 1.26+
    slog.NewJSONHandler(os.Stdout, nil),
    slog.NewTextHandler(logFile, nil),
))
logger.Info("request", slog.GroupAttrs("http", attrs...)) // 1.25+
```

---

## Toolchain

`go fix ./...` applies every modernizer below; `go fix -diff ./...` previews. Treat this
list as the checklist a reviewer will run:

`any` `atomictypes` `embedlit` `errorsastype` `forvar` `hostport` `inline` `mapsloop`
`minmax` `newexpr` `omitzero` `plusbuild` `rangeint` `reflecttypefor` `slicesbackward`
`slicescontains` `slicessort` `stditerators` `stringsbuilder` `stringscut`
`stringscutprefix` `stringsseq` `testingcontext` `unsafefuncs` `waitgroupgo`

(`waitgroup` was renamed `waitgroupgo` and `fmtappendf` was removed in 1.27.)

```go
// Deprecated: use NewFoo.
//
//go:fix inline
func OldFoo() *Foo { return NewFoo() } // go fix rewrites callers
```

Other toolchain behaviour worth relying on:

- `go test` runs the `stdversion` vet check by default (1.27): using an API newer than the
  `go` directive is now a test failure, not a runtime surprise.
- `go get -tool <pkg>` + `go tool <name>` replaces `tools.go`. (1.24)
- `go doc pkg@v1.2.3`, `go doc -ex pkg`, `go doc pkg.ExampleFoo` print versioned docs and
  example source. (1.27)
- `go mod tidy` collapses `require` blocks to two (direct, indirect) for `go 1.27` modules.
- `go tool trace -http=:6060` now binds localhost only; pass `0.0.0.0:6060` deliberately.
- `/debug/pprof/goroutineleak` (and `pprof.Lookup("goroutineleak")`) reports goroutines
  blocked on primitives no runnable goroutine can reach. (1.27)

---

## What each release added

- **1.22** per-iteration loop vars; `range` over int; `math/rand/v2`; `ServeMux` methods,
  `{param}`, `{path...}`, `{$}`; `slices.Concat`; `reflect.TypeFor`; `go/version`.
- **1.23** `iter` + range-over-func; `slices`/`maps` iterators; `unique`; GC-safe timers,
  no drain before `Reset`; `os.CopyFS`; `http.ParseCookie`.
- **1.24** generic type aliases; `weak`; `runtime.AddCleanup`; Swiss-table maps; `os.Root`;
  `b.Loop`, `t.Context`, `t.Chdir`; `slog.DiscardHandler`; `strings.Lines`/`SplitSeq`/
  `FieldsSeq`; `crypto/{sha3,hkdf,pbkdf2,mlkem}`; `rand.Text`; `NewGCMWithRandomNonce`;
  `omitzero`; `Server.Protocols`; tool directives; FIPS 140-3.
- **1.25** `synctest.Test`; container-aware `GOMAXPROCS`; `http.CrossOriginProtection`;
  `sync.WaitGroup.Go`; `trace.FlightRecorder`; `os.Root` write methods;
  `reflect.TypeAssert`; `t.Attr`/`t.Output`; `slog.GroupAttrs`; `io/fs.ReadLinkFS`.
- **1.26** `new(expr)`; `errors.AsType`; Green Tea GC on by default; `crypto/hpke`;
  reader-less crypto; `bytes.Buffer.Peek`; `reflect` type/value iterators;
  `slog.NewMultiHandler`; `t.ArtifactDir`; `go fix` modernizers; `ReverseProxy.Rewrite`.
- **1.27** generic methods; promoted-field literal keys; generalized function type
  inference; `uuid`; `encoding/json/v2` + `jsontext` GA; `crypto/mldsa`;
  `strings`/`bytes.CutLast`; `synctest.Sleep`; `httptest.NewTestServer`;
  `math/rand/v2 Rand.N`; `url.Clone`; `maphash.Hasher`; `big.Int.Divide`;
  goroutine-leak profile GA; Unicode 17.

---

## Free performance

Automatic -- do not hand-optimize around them.

- Maps: Swiss Tables since 1.24 (~30% faster; ~35% when pre-sized).
- `sync.Map`: hash-trie since 1.24 (~49% faster geomean).
- GC: Green Tea default since 1.26 (10--40% less GC overhead, more on Ice Lake / Zen 4+).
- cgo call overhead ~30% lower, `io.ReadAll` ~2x faster on ~half the memory, and
  `fmt.Errorf` matches `errors.New` for unformatted strings -- all since 1.26.
- Allocation: size-specialized malloc in 1.27 makes small (<80 B) allocations up to 30%
  cheaper, ~1% overall in allocation-heavy programs, for ~60 KB of binary size.
- `compress/flate` is faster in 1.27 -- and its **bytes changed**, so golden-file tests
  covering `gzip`, `zlib`, `zip` or `png` output may need regenerating. Same for
  `unicode`-dependent goldens (Unicode 15 -> 17).
