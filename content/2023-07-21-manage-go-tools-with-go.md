---
title: Manage Go tools with Go
date: 2023-07-21T17:00:00+02:00
hn_link: https://news.ycombinator.com/item?id=36814653
---

In almost any code, there comes a time when you need to use external tools for certain functionality. For example, you may want to use [`golangci-lint`](github.com/golangci/golangci-lint) to lint your code, generate mocks with [`mockgen`](https://github.com/golang/mock), or run your DB schema migrations with [`sql-migrate`](https://github.com/rubenv/sql-migrate).

There are few ways to achieve this: some projects use `Makefile` to install dependencies, some other use `docker compose` and run tools by running containers with mounted volumes.

Today, I want to share a method that is known within Go community, but is still not frequently used.

To understand how it works, let's start from scratch and install `golangci-lint` with Go:

```sh
go install github.com/golangci/golangci-lint/cmd/golangci-lint@latest
golangci-lint run
```

The first command installs the `latest` version of `golangci-lint` and the second command executes it (you will need to have `$GOPATH/bin` in your `PATH` env).

Of course, you can also install specific version of the binary:

```sh
go install github.com/golangci/golangci-lint/cmd/golangci-lint@v1.53.3
golangci-lint run
```

While the second option is better, as you use specific version of the tool, it's still cumbersome to use. For example, what happens when new version of `golangci-lint` comes out and you want to update? You and all your teammates will need to update the version manually. Even with `Makefile`, you will need to run a `make` command to install the latest version. This problem is even more prominent if you have multiple repositories that all use same tools (you will not always have time to update all of them at once).

This problem can be fixed with `docker` or `docker compose`, but Docker is slow and it doesn't integrate well with `go` commands.

Luckily, since [Go 1.17](https://go.dev/doc/go1.17#go%20run) `go run` command accepts an optional version suffifx, which lets us run any Go binary without installing it locally:

```sh
go run github.com/golangci/golangci-lint/cmd/golangci-lint@v1.53.3 run
```

This is already an improvement, because this line can now be put in e.g. `Makefile` and it will work for any repository letting the developers forget about the problems of keeping their local version in sync with the version needed by the project. One person can update the version and it will be automatically used by every team member once they checkout the main branch.

You can also use it with `go generate` command:

```go
//go:generate go run github.com/vektra/mockery/v2@v2.32.0 --name HTTPListener
type HTTPListener func(addr string, handler http.Handler) error
```

This is great, however for large repositories, it may be a bit repetitive. **`go run` downloads the code for the binary from the internet and just runs it, which could have massive security implications**. I like to improve from here by using an older pattern, or as I call it, "the `tools` pattern".

This is how it looks like in practice (this assumes this file is saved under `tools/tools.go`):

```go
//go:build tools

// To install everything from this file, run:
// go generate -tags tools tools/tools.go

package tools

import (
	_ "github.com/cosmtrek/air"                             //go:generate go install github.com/cosmtrek/air
	_ "github.com/daixiang0/gci"                            //go:generate go install github.com/daixiang0/gci
	_ "github.com/golangci/golangci-lint/cmd/golangci-lint" //go:generate go install github.com/golangci/golangci-lint/cmd/golangci-lint
	_ "github.com/vektra/mockery/v2"                        //go:generate go install github.com/vektra/mockery/v2
	_ "gotest.tools/gotestsum"                              //go:generate go install gotest.tools/gotestsum
	_ "mvdan.cc/gofumpt"                                    //go:generate go install mvdan.cc/gofumpt
)
```

Let me explain what happens. First, we add `//go:build tools` comment which instruments Go to include this file in a package **only** when `tools` tag is explicitly specified.

Then, we specify the package name `tools` and a list of imports. Each import path is aliased to `_` (because we don't actually use it in the file) and a path to the binary that you want to use. I also add `//go:generate` comment with `go install` command to install the binary in that line. Notice the `go install` command does not specify the version this time.

With this pattern, Go is smart enough to include these packages in `go.mod` file and thus version them. We can then vendor those, or validate that none of the packages have been modified (with `go mod verify`) which improves security and repeatability in CI. Because I also add `//go generate` comments, all tools can be installed with `go generate -tags tools tools/tools.go` line. Go knows which version to install. Have I mentioned I call it the `tools` pattern?

`go generate` commands can then skip the version:

```go
//go:generate go run github.com/vektra/mockery/v2 --name HTTPListener
type HTTPListener func(addr string, handler http.Handler) error
```

Now your Go tools can be installed and versioned by Go. Isn't that great?

## Caveats

It would be dishonest to not mention a few things that are not great about this approach:

1. `go.mod` and `go.sum` become polluted with dependencies.

   Depending on how much you care about keeping your dependency list clean, this may not be desired.

   If you prefer to not pollute the `go.mod` file, you still may use the `go run tool@version` approach and skip the `tools` pattern. It will let you version the dependencies, but is a bit more cumbersome to maintain.

2. You run self-built binary

   Because the dependency versions are resolved by `go.mod` file in projects repository, the locally compiled binary may by slightly different than the one officially distributed.

   I've personally never had any problems with it, but tools such as `golangci-lint` that heavily rely on dependency versions for functionality may behave slightly differently if you update packages only selectively. Personally, I've never experienced any problem like that.

   Some tools, even go as far as to print a warning. For example [PlanetScale's](https://planetscale.com/) CLI, [`pscale`](https://github.com/planetscale/cli), prints:

   ```txt
   !! WARNING: You are using a self-compiled binary which is not officially supported.
   !! To dismiss this warning, set PSCALE_DISABLE_DEV_WARNING=true
   ```

3. Doesn't work with non-Go binaries.

   Sometimes, you just need to have a tool that is not build with Go. For example, when using gRPC, [`protoc-gen-go`](https://github.com/golang/protobuf) is used which can be versioned with this method, however, its dependency, [`protoc`](https://grpc.io/docs/protoc-installation/) cannot, as it's not a Go program.

   Unfortunately, it means that for some repositories, you'd still need to maintain an alternative way to version binaries. Depending on the use case, it may still be worth it.

4. You need to configure your editor to use it.

   By default, most editors will not run `go run ...` for you so it requires a bit of configuration. The good news, is that once it's done, it will work for any repository that uses this pattern. For most tools, it's not required anyway.

## Summary

I really like this pattern as it allows me to forget about managing versions of the tools I use in my projects. Additionally, tools like [Dependabot](https://github.com/dependabot) or [Renovate](https://www.mend.io/renovate/) can help automate this process and keep your tools (and other dependencies) up to date. In short, I like that it "just works".

There's an [open proposal](https://github.com/golang/go/issues/48429) in Go's Github repository to track tool dependencies in `go.mod`. This proposal is meant to improve the experience of managing tool dependencies with Go, but it's unclear when, or how, it will be implemented.
