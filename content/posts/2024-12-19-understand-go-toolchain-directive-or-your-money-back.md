---
title: Understand Go toolchain directive or your money back
date: 2024-12-19
categories:
- blog
type:
- post
- posts
---
> With this compatibility support, the latest Go toolchain should always be the best, most secure implementation of an older version of Go.  
<https://go.dev/doc/go1.21#tools>

> You’ll never have to manually download and install a Go toolchain again. The go command will take care of it for you.
<https://go.dev/blog/toolchain>

Go 1.21 added a new `toolchain` directive to `go.mod`. I found it challenging to fully understand its behavior, since there are several lengthy docs on the subject.  
So here is a concise overview with examples:

`Toolchain` - the standard library as well as the compiler, assembler, and other tools.

1. ```go.mod
    go minimum-go-language-version
    ```

    <https://go.dev/doc/modules/gomod-ref#go>

    The `go` directive in `go.mod` indicates that a module was written assuming the semantics of a given version of Go.  
    It affects the use of new language features, e.g. [loopvar behavior](https://go.dev/blog/go1.22#language-changes),
    availability of [`math/rand/v2`](https://go.dev/blog/go1.22#standard-library-additions), etc...

    Per the [release policy](https://go.dev/doc/devel/release#policy), critical problems are fixed by issuing patch releases.  
    To get [fixes](https://go.dev/doc/devel/release#go1.23.minor) that are released in patch revisions, one needs to use the newer toolchain.

    The version must be a valid Go version, such as 1.20, 1.22.0, or 1.24rc1.  
    <https://go.dev/ref/mod#go-mod-file-go>

    Note that released versions of Go use the version syntax ‘1.N.P’, denoting the Pth release of Go 1.N.

    The syntax `1.N` (e.g. `1.22`) is called a "language version." It denotes the overall family of Go releases implementing that version of the Go language and standard library.  
    When comparing two Go versions within a language version, the ordering from least to greatest is as follows:  
    For example, 1.21 < 1.21rc1 < 1.21rc2 < 1.21.0 < 1.21.1 < 1.21.2.  
    <https://go.dev/doc/toolchain#version>

    Although there are usually no new language features in patch releases, to indicate that your module needs a _released_ version, you must include the patch release, e.g.:

    ```
    go 1.23.0
    ```

    This is confirmed by a member of the Go team here: <https://github.com/golang/go/issues/68971#issuecomment-2300236006>

    Also, we can look at what others do:  
    [127k files](https://github.com/search?q=%28%2Fgo+1.2%5B1-9%5D%24%2F%29+path%3A**%2Fgo.mod++NOT+is%3Aarchived&type=code&ref=advsearch) do not specify the patch release,  
    and [305k files](https://github.com/search?q=%28%2Fgo+1%5C.2%5B1-9%5D%5C.%5B0-9%5D%2B%24%2F%29+path%3A**%2Fgo.mod++NOT+is%3Aarchived&type=code&ref=advsearch) do.

2. ```go.mod
    toolchain minimum-toolchain-to-use
    ```

    The `toolchain` directive in `go.mod` specifies **the minimum** Go toolchain to use when working in a particular module.

    Go toolchains are named `goV`, where V is a Go version denoting a release or release candidate.
    For example, `go1.23.0` or `go1.24rc1`.  
    <https://go.dev/doc/toolchain#name>

    If the toolchain line is omitted, the module or workspace is considered to have an implicit `toolchain goV` line, where V is the Go version from the go line.  
    So these `go.mod` files would be equivalent:

    ```
    go 1.23.0
    ```

    and

    ```
    go 1.23.0
    toolchain go1.23.0
    ```

    The go command selects the Go toolchain to use based on the `GOTOOLCHAIN` setting.

    * `GOTOOLCHAIN=auto` – This is the default.
      * The `go` command uses its own bundled toolchain when that toolchain is **at least as new as the go or toolchain lines in the main module** or workspace.
      * When the go or toolchain line is newer than the bundled toolchain, the go command **downloads and uses the specified toolchain** instead.  
      These toolchains are packaged as special modules with the module path `golang.org/toolchain` and version `v0.0.1-goVERSION.GOOS-GOARCH`. Toolchains are downloaded like any other module.  

      So that means, for example, if your `go.mod` specifies `toolchain 1.23.5`, but you have Go `1.24.0` installed, your binary will be built using the `1.24` toolchain.  
      I feel like this default behavior violates the [Principle of least astonishment](https://en.wikipedia.org/wiki/Principle_of_least_astonishment) and makes it harder to achieve [hermetic](https://abseil.io/resources/swe-book/html/ch18.html#:~:text=Tools%20as%20dependencies,to%20each%20target)
      [builds](https://bazel.build/basics/hermeticity#:~:text=Isolation%3A%20Hermetic%20build%20systems%20treat%20tools%20as%20source%20code.%20They%20download%20copies%20of%20tools%20and%20manage%20their%20storage%20and%20use%20inside%20managed%20file%20trees.%20This%20creates%20isolation%20between%20the%20host%20machine%20and%20local%20user%2C%20including%20installed%20versions%20of%20languages).  
      But, luckily, there are other settings to choose from:

    * `GOTOOLCHAIN=local` – automatic downloads are disabled. The local Go toolchain is used if it is >= `go` or `toolchain` directive in `go.mod`. Otherwise, the module fails to build.
    * `GOTOOLCHAIN=toolchain_version` (e.g. `GOTOOLCHAIN=go1.23.0`) – the go command always runs that specified Go toolchain.  
      If a binary with that name is found in the system PATH, the go command uses it. Otherwise, the go command uses a Go toolchain it downloads and verifies.

    <https://go.dev/doc/toolchain>  
    <https://go.dev/blog/toolchain>

3. Misc

    The `go` and `toolchain` requirements can be updated using `go get` like ordinary module requirements.  
    `go get go@1.21.0`  
    `go get toolchain@go1.21.0`

    The special form `toolchain@none` means to remove any toolchain line, as in  
    `go get toolchain@none`  
    or  
    `go get go@1.25.0 toolchain@none`  

    `go get go@latest` updates the module to require the latest released Go toolchain.

4. Additional links  
    Discussion in GitHub CLI repo: <https://github.com/cli/cli/issues/9489>  

    Comment from Russ Cox on the default `GOTOOLCHAIN` value in a Docker image:
    <https://github.com/docker-library/golang/issues/472#issuecomment-1721760993>

    <https://go.dev/ref/mod>  
    <https://go.dev/doc/modules/gomod-ref>  
    <https://go.dev/doc/godebug>  

    <https://www.youtube.com/watch?v=v24wrd3RwGo>  
    <https://go.googlesource.com/proposal/+/master/design/56986-godebug.md>  
