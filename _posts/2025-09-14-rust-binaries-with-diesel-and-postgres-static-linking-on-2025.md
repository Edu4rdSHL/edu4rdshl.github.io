---
layout: post
title: 'Rust binaries with Diesel and Postgres: static linking on 2025'
date: 2025-09-14 19:36 -0500
categories: ["rust", "diesel", "postgres", "static linking"]
tags: ["rust", "diesel", "postgres", "static linking", "musl", "pq-sys"]
author: edu4rdshl
image:
  path: /2025-09-14/static-rust-and-diesel.png
  alt: A image of Rust and Diesel logos
excerpt: A small, painless, and straightforward guide on how to statically link Rust binaries that use Diesel and Postgres
---
## TLDR

See the [summarizing](#summarizing) section.

## Introduction

I have been writing Rust code in a professional manner since 2019. The "de-facto" ORM for Rust on that time was [Diesel](https://diesel.rs/), and I have been using it since then. The software that I was writing was meant to run on almost any Linux system, while being distributed as pre-compiled binaries. The easiest way to achieve that is to statically link the binaries, so they don't depend on any shared libraries that may or may not be present on the target system.

## The problem

Rust is really good at static linking. Avoiding the (g)libc problem is as easy as using [musl](https://musl.libc.org/) as the target. For OpenSSL, it's as easy as using the [vendored](https://docs.rs/openssl/latest/openssl/#vendored) feature. But the projects that I was working on, also used [Diesel](https://diesel.rs/) with [Postgres](https://www.postgresql.org/) as the database backend. Diesel uses [pq-sys](https://crates.io/crates/pq-sys) crate as the low-level binding to the Postgres C client library, `libpq`. And `libpq` is a nightmare when it comes to static linking due to its hard dependencies on other shared libraries, like OpenSSL, `libssl`, `libcrypto`, etc., so you need to bundle **all** of their static files built for your target.

That's why custom docker containers like [rust-musl-builder](https://github.com/emk/rust-musl-builder), [rust_musl_docker](https://gitlab.com/rust_musl_docker/image), and others, were created. They provide a pre-configured environment with musl, OpenSSL, Postgres, and libpq statically linked, so you can build your Rust binaries with Diesel and Postgres support. They were hard to maintain, and often broke due to changes in the underlying libraries; as a result, they were abandoned by their maintainers and CI/CD pipelines that used them broke.

**Note**: yes, I know that there's [sqlx](https://crates.io/crates/sqlx), which uses [rustls](https://crates.io/crates/rustls) instead of OpenSSL, and can be statically linked without any issues. But when you have more than 30 different projects using Diesel for many things, migrating them all to sqlx is not a small task.

## The solution

As I mentioned before, there used to exist some docker images that provided a pre-configured environment for building Rust binaries with Diesel and Postgres support, big kudos to their maintainers, they worked very well for many years. After the CI/CD fails, I started to investigate if there was another way to achieve the same result without resorting to forking and maintaining a custom docker image.

As mentioned above, the main problem is `libpq`. You can spin up an Alpine Linux container, install the required dependencies (`build-base pkgconfig postgresql-dev libbsd-dev openssl-dev zlib-dev zlib-static perl git` in my case), but you will still hit the `libpq` wall:

{ % raw % }
```
error: linking with `cc` failed: exit status: 1
  |
  = note:  "cc" "-m64" "<sysroot>/lib/rustlib/x86_64-unknown-linux-musl/lib/self-contained/rcrt1.o" "<sysroot>/lib/rustlib/x86_64-unknown-linux-musl/lib/self-contained/crti.o" "<sysroot>/lib/rustlib/x86_64-unknown-linux-musl/lib/self-contained/crtbeginS.o" "/tmp/rustchvPyBK/symbols.o" "<1 object files omitted>" "-Wl,--as-needed" "-Wl,-Bstatic" "-lpq" "/tmp/rustchvPyBK/{libzstd_sys-6c0886af15f7a887.rlib,libopenssl_sys-ccd5a7f01478a311.rlib}.rlib" "-lunwind" "-lc" "<sysroot>/lib/rustlib/x86_64-unknown-linux-musl/lib/{libcompiler_builtins-*}.rlib" "-L" "/tmp/rustchvPyBK/raw-dylibs" "-Wl,-Bdynamic" "-Wl,--eh-frame-hdr" "-Wl,-z,noexecstack" "-nostartfiles" "-L" "/build/target/x86_64-unknown-linux-musl/release/build/zstd-sys-ba1068f06e2ad027/out" "-L" "/build/target/x86_64-unknown-linux-musl/release/build/openssl-sys-8fb0ce292de28293/out/openssl-build/install/lib" "-L" "/usr/lib" "-L" "<sysroot>/lib/rustlib/x86_64-unknown-linux-musl/lib/self-contained" "-L" "<sysroot>/lib/rustlib/x86_64-unknown-linux-musl/lib" "-o" "/build/target/x86_64-unknown-linux-musl/release/deps/test_api-fd10fed5f8242f8f" "-Wl,--gc-sections" "-static-pie" "-Wl,-z,relro,-z,now" "-Wl,-O1" "-Wl,--strip-all" "-nodefaultlibs" "<sysroot>/lib/rustlib/x86_64-unknown-linux-musl/lib/self-contained/crtendS.o" "<sysroot>/lib/rustlib/x86_64-unknown-linux-musl/lib/self-contained/crtn.o"
  = note: some arguments are omitted. use `--verbose` to show all linker arguments
  = note: /usr/lib/gcc/x86_64-alpine-linux-musl/14.2.0/../../../../x86_64-alpine-linux-musl/bin/ld: /usr/lib/libpq.a(fe-connect.o): in function `defaultNoticeProcessor':
          fe-connect.c:(.text+0x6a8): undefined reference to `pg_fprintf'
          /usr/lib/gcc/x86_64-alpine-linux-musl/14.2.0/../../../../x86_64-alpine-linux-musl/bin/ld: /usr/lib/libpq.a(fe-connect.o): in function `sslVerifyProtocolVersion':
          fe-connect.c:(.text+0x6d1): undefined reference to `pg_strcasecmp'
          /usr/lib/gcc/x86_64-alpine-linux-musl/14.2.0/../../../../x86_64-alpine-linux-musl/bin/ld: fe-connect.c:(.text+0x6e5): undefined reference to `pg_strcasecmp'
          /usr/lib/gcc/x86_64-alpine-linux-musl/14.2.0/../../../../x86_64-alpine-linux-musl/bin/ld: fe-connect.c:(.text+0x6f9): undefined reference to `pg_strcasecmp'
          /usr/lib/gcc/x86_64-alpine-linux-musl/14.2.0/../../../../x86_64-alpine-linux-musl/bin/ld: fe-connect.c:(.text+0x70d): undefined reference to `pg_strcasecmp'
          /usr/lib/gcc/x86_64-alpine-linux-musl/14.2.0/../../../../x86_64-alpine-linux-musl/bin/ld: /usr/lib/libpq.a(fe-connect.o): in function `emitHostIdentityInfo':
          fe-connect.c:(.text+0x845): undefined reference to `pg_getnameinfo_all'
          /usr/lib/gcc/x86_64-alpine-linux-musl/14.2.0/../../../../x86_64-alpine-linux-musl/bin/ld: /usr/lib/libpq.a(fe-connect.o): in function `connectFailureMessage':
          fe-connect.c:(.text+0x988): undefined reference to `pg_strerror_r'
...snip
          collect2: error: ld returned 1 exit status
          
  = note: some `extern` functions couldn't be found; some native libraries may need to be installed or have their path specified
  = note: use the `-l` flag to specify native libraries to link
  = note: use the `cargo:rustc-link-lib` directive to specify the native libraries to link with Cargo (see https://doc.rust-lang.org/cargo/reference/build-scripts.html#rustc-link-lib)

error: could not compile `test_api` (bin "test_api") due to 1 previous error
Error: building at STEP "RUN cargo build --target x86_64-unknown-linux-musl --release --locked": while running runtime: exit status 101
```
{ % endraw % }

### pq-sys 0.5.0

[pq-sys 0.5.0](https://github.com/sgrif/pq-sys/releases/tag/v0.5.0) was released in early 2024, with a **VERY** handy feature:

- We added a `pq-src` crate and a `bundled` feature for `pq-sys`. This allows to build and link a static version of libpq during the rust build process. This feature currently supports builds targeting Windows, Linux and macOS. It requires a c-compiler toolchain for the target to build libpq from source.

This solved all the headaches of dealing with `libpq` and its dependencies, as now you can just enable the `bundled` feature and everything builds just fine on an Alpine Linux container. I was really surprised that this solution is not found in the Diesel documentation, and almost nowhere on the internet.

### Summarizing

So, summarizing, to build Rust binaries with Diesel and Postgres support, statically linked, you need something like this on your `Cargo.toml`:

```toml
[dependencies]
diesel = { version = "2.3.1", features = [
    "postgres",
    "r2d2",
    "time",
    "chrono",
    "uuid",
    "numeric",
    "network-address",
] }
#### Needed to build static binaries ####
pq-sys = { version = "0.7.2", features = ["bundled"] }
openssl-sys = { version = "0.9.109", features = ["vendored"] }
```

And then build your project targeting `x86_64-unknown-linux-musl`:

```sh
cargo build --target x86_64-unknown-linux-musl --release --locked
```

That's it! You will get a fully statically linked binary that works on almost any Linux system.

## Notes

According to the Diesel maintainers, the reason this solution is almost nowhere to be found is because it has only been implemented for a _short_ time, compared to Diesel's existence, which dates back to 2016.

## Conclusion

I hope this small guide helps you to build your Rust binaries with Diesel and Postgres support, statically linked, without the need to maintain custom docker images or deal with the headaches of `libpq` and its dependencies. I will be sending a PR to the Diesel documentation to include this information, as I believe it will help many people in the Rust community.