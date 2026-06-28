---
title: go-module
---

`go-module` is the gomatic ecosystem's **module-identity parser** for Go. It turns a git remote URL into a Go module [`Path`](https://pkg.go.dev/github.com/gomatic/go-module#Path) and the name variants derived from it — the repository [`Name`](https://pkg.go.dev/github.com/gomatic/go-module#Name), a Go [`Identifier`](https://pkg.go.dev/github.com/gomatic/go-module#Identifier), and an [`EnvPrefix`](https://pkg.go.dev/github.com/gomatic/go-module#EnvPrefix). It is pure string logic with no I/O — a reusable leaf library for any caller that needs to reason about a project's identity from its remote.

- **Source:** [`gomatic/go-module`](https://github.com/gomatic/go-module)
- **API reference:** [pkg.go.dev/github.com/gomatic/go-module](https://pkg.go.dev/github.com/gomatic/go-module)

## Install

```sh
go get github.com/gomatic/go-module
```

## What it parses

[`Parse`](https://pkg.go.dev/github.com/gomatic/go-module#Parse) accepts the scp-like SSH form (`git@host:org/repo.git`) and URL forms (`https://host/org/repo.git`, `ssh://git@host/org/repo`). It strips the scheme, any userinfo, and a trailing `.git`, then validates that the result is a `host/org/repo` path — at least three non-empty, space-free segments. When the result is not such a path it returns [`ErrInvalidRemote`](https://pkg.go.dev/github.com/gomatic/go-module#pkg-constants).

The package owns the **derivation only** — it performs no network or filesystem access. Each of the parsed values is its own named type, so callers never confuse a remote with a path or a name with its identifier:

```go
import module "github.com/gomatic/go-module"

// Remote -> Path -> Name -> Identifier / EnvPrefix
```

## Usage

### Parse a remote

```go
package main

import (
	"fmt"

	module "github.com/gomatic/go-module"
)

func main() {
	path, err := module.Parse(module.Remote("git@github.com:org/repo.git"))
	if err != nil {
		panic(err)
	}

	fmt.Println(path) // github.com/org/repo
}
```

### Derive the name variants

[`Path.Repo`](https://pkg.go.dev/github.com/gomatic/go-module#Path.Repo) returns the last path segment as a [`Name`](https://pkg.go.dev/github.com/gomatic/go-module#Name); [`Name.Identifier`](https://pkg.go.dev/github.com/gomatic/go-module#Name.Identifier) and [`Name.EnvPrefix`](https://pkg.go.dev/github.com/gomatic/go-module#Name.EnvPrefix) reduce it to a Go identifier and an environment-variable prefix:

```go
path, _ := module.Parse(module.Remote("git@github.com:gomatic/template.cli.git"))

name := path.Repo()

fmt.Println(path)              // github.com/gomatic/template.cli
fmt.Println(name)              // template.cli
fmt.Println(name.Identifier()) // templatecli
fmt.Println(name.EnvPrefix())  // TEMPLATE_CLI
```

[`Identifier`](https://pkg.go.dev/github.com/gomatic/go-module#Name.Identifier) lowercases the name and keeps only lowercase letters and digits, dropping every other character. [`EnvPrefix`](https://pkg.go.dev/github.com/gomatic/go-module#Name.EnvPrefix) uppercases the name and maps every non-alphanumeric character to an underscore.

### Handle an invalid remote

`Parse` returns [`ErrInvalidRemote`](https://pkg.go.dev/github.com/gomatic/go-module#pkg-constants), matchable with [`errors.Is`](https://pkg.go.dev/errors#Is), when the remote does not resolve to a `host/org/repo` path:

```go
_, err := module.Parse(module.Remote("https://example.com/onlyone"))
fmt.Println(errors.Is(err, module.ErrInvalidRemote)) // true
```

Userinfo is stripped before the remote is echoed into the error, so a `user:token@` credential can never leak into the error text.

## Design

- **Every parsed value is a named string type** — [`Remote`](https://pkg.go.dev/github.com/gomatic/go-module#Remote), [`Path`](https://pkg.go.dev/github.com/gomatic/go-module#Path), [`Name`](https://pkg.go.dev/github.com/gomatic/go-module#Name), [`Identifier`](https://pkg.go.dev/github.com/gomatic/go-module#Identifier), [`EnvPrefix`](https://pkg.go.dev/github.com/gomatic/go-module#EnvPrefix) — so the type system distinguishes the stages of the pipeline and values are safe to copy and compare.
- **Pure and I/O-free** — `Parse` and its derivations are total string functions; there is no network or filesystem access, which keeps the package a fast, deterministic, testable leaf.
- **Credentials never leak** — `Parse` strips any `user@` / `user:token@` userinfo before validating, so an invalid remote's error text cannot expose a secret.
- **Constant sentinel error** — `ErrInvalidRemote` is a [`gomatic/go-error`](https://github.com/gomatic/go-error) constant, matched with `errors.Is`, never by string.

## Who uses it

gomatic tooling derives a project's identity from its remote with `go-module`: the [`gomatic/template.cli`](https://github.com/gomatic/template.cli) scaffold and the rename tooling that retargets a generated project both lean on the same `Path` / `Name` / `Identifier` / `EnvPrefix` derivation.
