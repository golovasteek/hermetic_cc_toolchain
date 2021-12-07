[![builds.sr.ht status](https://builds.sr.ht/~motiejus/bazel-zig-cc.svg)](https://builds.sr.ht/~motiejus/bazel-zig-cc)

# Bazel zig cc toolchain

This is a C/C++ toolchain that can (cross-)compile C/C++ programs. It contains
clang-12, musl, glibc (versions 2-2.33, selectable), all in a ~40MB package.
Read
[here](https://andrewkelley.me/post/zig-cc-powerful-drop-in-replacement-gcc-clang.html)
about zig-cc; the rest of the README will present how to use this toolchain
from Bazel.

# Usage

Add this to your `WORKSPACE`:

```
BAZEL_ZIG_CC_VERSION = "v0.3.3"

http_archive(
    name = "bazel-zig-cc",
    sha256 = "0bf4ea0ceaa3a2e3be6e891e6c0035d4c86e4cc69ccc1d7ddf701d4c1e4c45a3",
    strip_prefix = "bazel-zig-cc-{}".format(BAZEL_ZIG_CC_VERSION),
    urls = ["https://git.sr.ht/~motiejus/bazel-zig-cc/archive/{}.tar.gz".format(BAZEL_ZIG_CC_VERSION)],
)

load("@bazel-zig-cc//toolchain:defs.bzl", zig_register_toolchains = "register_toolchains")

zig_register_toolchains(register = [
    "x86_64-linux-gnu.2.28",
    "x86_64-macos-gnu",
])
```

The snippet above will download the zig toolchain and register it for the
following platforms:

- `x86_64-linux-gnu.2.28` for `["@platforms//os:linux", "@platforms//cpu:x86_64"]`.
- `x86_64-macos-gnu` for `["@platforms//os:macos", "@platforms//cpu:x86_64"]`.


Note that both Go and Bazel naming schemes are accepted. For convenience with
Go, the following Go-style toolchain aliases are created:

|Bazel (zig) name |Go name|
--- | ---
|`x86_64`|`amd64`|
|`aarch64`|`arm64`|
|`macos`|`darwin`|

For example, the toolchain `linux_amd64_gnu` is aliased to
`x86_64-linux-gnu.2.28`. To find out which toolchains can be registered or
used, run:

```
$ bazel query @zig_sdk//... | sed -En '/.*_toolchain$/ s/.*:(.*)_toolchain$/\1/p'
```

Read [#Known Issues](#known-issues) before using.

# Known Issues

## Parallel `zig c++` invocations may fail

**Severity: High**

**Task:** [ziglang/zig #9431 FileNotFound when compiling macos](https://github.com/ziglang/zig/issues/9431)

Background: there is a race when calling `zig c++`, which Bazel does a lot.
This may fail compilation.

## relocation error with glibc < 2.32

**Severity: High**

**Task:** [ziglang/zig relocation error: symbol pthread_sigmask version GLIBC_2.2.5 not defined in file libc.so.6 with link time reference #7667](https://github.com/ziglang/zig/issues/7667)

Background: one of our internal shared libraries (which we must build with glibc 2.19) does not load on an older system:

```
id: relocation error: /lib/x86_64-linux-gnu/libnss_uber.so.2: symbol pthread_sigmask, version GLIBC_2.2.5 not defined in file libc.so.6 with link time reference
```

Severity is high, because there is no known workaround: the shared library,
when built with this toolchain, will not work on our target system.

## newer zig fails darwin tests

**Severity: Low**

Task: [ziglang/zig #9485 glibc 2.27 or older: fcntl64 not found, but zig's glibc headers refer it](https://github.com/ziglang/zig/issues/9485)

Background: when glibc 2.27 or older is selected, it may miss `fcntl64`. A
workaround is applied for `x86_64`, but not for aarch64. The same workaround
may apply to aarch64, but the author didn't find a need to test it (yet).

## incorrect glibc version autodetection

**Severity: Low**

**Task:** [ziglang/zig zig detects wrong libc version #6469](https://github.com/ziglang/zig/issues/6469)

Background: zig detects an incorrect glibc version when not specified.
Therefore, until the task is resolved, registering a GNU toolchain without a
version suffix (e.g. `linux_amd64_gnu`) is not recommended. We recommend
specifying the suffix to the oldest system that is mean to run the compiled
binaries. This is safe, because glibc is backwards-compatible. Alternatively,
use musl.

# Closed issues

- [ziglang/zig #9139 zig c++ hanging when compiling in parallel](https://github.com/ziglang/zig/issues/9139) (CLOSED, thanks andrewrk)
- [golang/go #46644 cmd/link: with CC=zig: SIGSERV when cross-compiling to darwin/amd64](https://github.com/golang/go/issues/46644) (CLOSED, thanks kubkon)
- [ziglang/zig #9050 golang linker segfault](https://github.com/ziglang/zig/issues/9050) (CLOSED, thanks kubkon)
- [ziglang/zig #7917 [meta] better c/c++ toolchain compatibility](https://github.com/ziglang/zig/issues/7917) (CLOSED, thanks andrewrk)
- [ziglang/zig #7915 ar-compatible command for zig cc](https://github.com/ziglang/zig/issues/7915) (CLOSED, thanks andrewrk)
- [rules/go #2894 Per-arch_target linker flags](https://github.com/bazelbuild/rules_go/issues/2894) (CLOSED, thanks mjonaitis)

# Testing

## linux cgo + glibc 2.19

```
$ bazel build --platforms @io_bazel_rules_go//go/toolchain:linux_amd64_cgo //test:hello
$ file bazel-out/k8-fastbuild-ST-d17813c235ce/bin/test/hello_/hello
bazel-out/k8-fastbuild-ST-d17813c235ce/bin/test/hello_/hello: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 2.0.0, Go BuildID=redacted, with debug_info, not stripped
```

## linux cgo + musl

```
$ bazel build \
    --platforms @io_bazel_rules_go//go/toolchain:linux_amd64_cgo \
    --extra_toolchains @zig_sdk//:linux_amd64_musl_toolchain //test:hello
...
$ file ../bazel-out/k8-fastbuild-ST-d17813c235ce/bin/test/hello_/hello
../bazel-out/k8-fastbuild-ST-d17813c235ce/bin/test/hello_/hello: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, Go BuildID=redacted, with debug_info, not stripped
$ ../bazel-out/k8-fastbuild-ST-d17813c235ce/bin/test/hello_/hello
hello, world
```

## macos cgo

```
$ bazel build --platforms @io_bazel_rules_go//go/toolchain:darwin_amd64_cgo //test:hello
...
$ file bazel-bin/test/hello_/hello
bazel-bin/test/hello_/hello: Mach-O 64-bit x86_64 executable, flags:<NOUNDEFS|DYLDLINK|TWOLEVEL|PIE>
```

## Transient docker environment

```
$ docker run -e CC=/usr/bin/false -ti --rm -v $(pwd):/x -w /x debian:bullseye-slim
# apt update && apt install direnv git -y
# . .envrc
```

And run the `bazel build` commands above. Take a look at `.build.yml` and see
how CI does it.

# Contribution guidelines

Contributions are accepted via patches to the mailing list
[~motiejus/bazel-zig-cc@lists.sr.ht][mailing-list]. A few ways to send patches:

1. `git send-email(1)`. More info at [git-send-email.io][git-send-email].
2. Sourcehut web UI. See [video][video] by sourcehut's creator Drew DeVault.

Copyright is retained by the contributors.

# Credits

Many thanks to Adam Bouhenguel and his [bazel-zig-cc][ajbouh], the parent of
this repository.

[mailing-list]: mailto:~motiejus/bazel-zig-cc@lists.sr.ht
[ajbouh]: https://github.com/ajbouh/bazel-zig-cc/
[git-send-email]: https://git-send-email.io/
[video]: https://spacepub.space/w/no6jnhHeUrt2E5ST168tRL
