# go-latest-version

A [skills.sh](https://skills.sh) context provider enforcing modern Go standards.

It keeps an LLM writing Go the way the current toolchain wants: modern language features,
current standard library APIs, no deprecated patterns, and correct version gating against
the `go` directive in `go.mod`.

## Supported Versions

| Version | Status |
| :--- | :--- |
| **Go 1.22** | Supported |
| **Go 1.23** | Supported |
| **Go 1.24** | Supported |
| **Go 1.25** | Supported |
| **Go 1.26** | Supported |
| **Go 1.27** | Supported |

## Contents

| File | Purpose |
| :--- | :--- |
| `SKILL.md` | Always loaded: rules, old-to-new tables, core snippets, toolchain |
| `references/MODERN-PATTERNS.md` | Worked examples per topic |
| `references/DEPRECATED.md` | Deprecations, removals, GODEBUG, GOEXPERIMENT, platforms |

## Usage

**skills.sh CLI**

```bash
npx skills add FumingPower3925/go-latest-version
```
