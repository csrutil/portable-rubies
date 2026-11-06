# portable-rubies

Prebuilt, relocatable Ruby for x86_64 Linux. Extract a tarball into any
directory and run it. You do not need root, a compiler, `LD_LIBRARY_PATH`,
or Ruby's library dependencies on the host.

Each Ruby release comes in three CPU tiers. Each tier compiles Ruby and all
bundled libraries for that microarchitecture level.

## Packages

| Ruby | Tiers |
|---|---|
| 4.0.9 | `generic`, `x86_64-v3`, `x86_64-v4` |
| 3.4.12 | `generic`, `x86_64-v3`, `x86_64-v4` |

Download packages from [Releases](../../releases). Each release tag is the
Ruby version, for example `4.0.9`. Every tarball has a `.sha256` file.

### Requirements

- x86_64 Linux with glibc 2.39 or newer, for example Ubuntu 24.04+ or
  Debian 13+. Packages do not run on musl (Alpine).
- A CPU that supports the tier you choose.

### Choose a tier

| Tier | Compiler flags | CPUs |
|---|---|---|
| `generic` | `-march=x86-64 -mtune=generic` | All x86_64 CPUs |
| `x86_64-v3` | `-march=x86-64-v3 -mtune=generic` | AVX2 class: Intel Haswell (2013) and newer, AMD Zen and newer |
| `x86_64-v4` | `-march=x86-64-v4 -mtune=generic` | AVX-512 class: Intel Xeon Skylake-SP and newer, Intel Ice Lake, AMD Zen 4 and newer |

To find the levels your CPU supports, ask the dynamic loader:

```sh
/lib64/ld-linux-x86-64.so.2 --help | grep supported
```

Use the highest tier marked `supported`. A tier above your CPU's level
crashes with `Illegal instruction`.

## Install

```sh
version=4.0.9
tier=x86_64-v3
name="ruby-${version}-linux-${tier}"
base="https://github.com/csrutil/portable-rubies/releases/download/${version}"

curl -fLO "${base}/${name}.tar.gz"
curl -fLO "${base}/${name}.tar.gz.sha256"
sha256sum -c "${name}.tar.gz.sha256"

sudo mkdir -p /opt/ruby
sudo tar -xzf "${name}.tar.gz" -C /opt/ruby
export PATH="/opt/ruby/${name}/bin:$PATH"

ruby -v
```

You can extract to any directory, for example `~/.local/share/ruby`. You can
also move the directory after you extract it.

## What is inside

Ruby is configured with:

```
--enable-shared --enable-load-relative --enable-yjit --with-jemalloc
--with-out-ext=win32,win32ole,-test-
--disable-install-doc --disable-install-rdoc
```

Bundled libraries, built from source with the tier's flags:

| Library | Version |
|---|---|
| OpenSSL | 3.0.22 |
| zlib | 1.3.1 |
| libyaml | 0.2.5 |
| libffi | 3.4.8 |
| GMP | 6.3.0 |
| jemalloc | 5.3.0 |
| libxcrypt | 4.4.38 |

The packages also include `libgcc_s` and `libstdc++` from the build
toolchain. Every ELF file has an `$ORIGIN`-relative `RUNPATH`, so it finds
the bundled libraries wherever the directory is.

### OpenSSL and certificates

`bin/ruby` is a small shell wrapper around `bin/ruby.bin`. It sets these
variables if you have not set them already:

- `SSL_CERT_FILE`: the first system CA bundle that exists among
  `/etc/ssl/certs/ca-certificates.crt` (Debian, Ubuntu, Arch),
  `/etc/pki/tls/certs/ca-bundle.crt` (Fedora, RHEL),
  `/etc/pki/ca-trust/extracted/pem/tls-ca-bundle.pem`,
  `/etc/ssl/ca-bundle.pem` (openSUSE), and `/etc/ssl/cert.pem`.
- `SSL_CERT_DIR`: `/etc/ssl/certs` or `/etc/pki/tls/certs`.
- `OPENSSL_CONF`: the bundled `ssl/openssl.cnf`, which loads the default
  provider only. The package does not read the distribution's OpenSSL
  config.
- `OPENSSL_MODULES`: the bundled `lib/ossl-modules`.

To use other values, set the variables before you run Ruby.

### Gems with native extensions

`gem install` works for gems with C extensions if the host has a C compiler
(for example `build-essential`). Extensions are compiled with the tier's
`-march` flags from `RbConfig`.

## Build locally

Build on x86_64 Linux. The glibc version of the build host sets the minimum
glibc version of the package.

```sh
sudo apt-get install autoconf bison build-essential curl patchelf \
  pkg-config rustc cargo xz-utils

scripts/build-portable-ruby x86_64-v3 4.0.9
ls dist/
```

The script downloads all sources and checks their SHA256 sums. It builds
the libraries and Ruby into `build/` and writes the tarball to `dist/`.
For `generic` and `x86_64-v3`, it also copies the result to a new
directory, runs Ruby there, and makes a TLS connection to rubygems.org.

### x86_64-v4 on hosts without AVX-512

The `x86_64-v4` build never runs v4 code on the build host. It first builds
a generic Ruby from the same source, and the build uses it for every step
that runs Ruby: code generation, extension configuration, and `make install`.
GMP is configured in cross mode, so it does not run test programs. For this
reason, CI only checks v4 packages statically and does not run them.

## Releases

The **Build and release** workflow runs when you start it from the Actions
tab. It builds all six packages on Ubuntu 24.04. When all builds pass, it
creates or updates one release per Ruby version, tagged with that version.

To add a Ruby version:

1. Add the version and the SHA256 of its `.tar.gz` (from
   <https://cache.ruby-lang.org/pub/ruby/index.txt>) to the `case` block at
   the top of `scripts/build-portable-ruby`.
2. Add the version to both `ruby` matrices in
   `.github/workflows/build.yml`.

## License

Ruby and the bundled libraries keep their own licenses: the Ruby License
or BSD-2-Clause for Ruby, Apache-2.0 for OpenSSL, LGPL-3.0-or-later or
GPL-2.0-or-later for GMP, LGPL-2.1-or-later for libxcrypt, BSD-2-Clause for
jemalloc, MIT for libyaml and libffi, and the zlib license for zlib.
