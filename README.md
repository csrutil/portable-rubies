# portable-rubies

Prebuilt, relocatable Ruby for x86_64 Linux. Extract a tarball into any
directory and run it. You do not need root, a compiler, `LD_LIBRARY_PATH`,
or Ruby's library dependencies on the host.

Each Ruby release comes in three CPU tiers. Each tier compiles Ruby, YJIT,
and all bundled libraries for that microarchitecture level. C and C++ code
uses `-O3` and GCC link-time optimization. YJIT uses release-mode ThinLTO.

## Packages

CI builds the latest two stable releases listed by
[ruby/ruby](https://github.com/ruby/ruby/releases). Each release includes the
`generic`, `x86_64-v3`, and `x86_64-v4` tiers.

Download packages from [Releases](../../releases). Each release tag is the
Ruby version, for example `4.0.9`. Every tarball has a `.sha256` file.

### Requirements

- x86_64 Linux with glibc 2.39 or newer, for example Ubuntu 24.04+ or
  Debian 13+. Packages do not run on musl (Alpine).
- A CPU that supports the tier you choose.

### Choose a tier

| Tier | ISA flags | CPUs |
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

Ruby builds YJIT but does not enable it by default. Enable it for long-running
workloads with `ruby --yjit`, `RUBY_YJIT_ENABLE=1`, or
`RubyVM::YJIT.enable` after application startup.

Bundled libraries, built from source with the tier's flags:

| Library | Version |
|---|---|
| OpenSSL | 4.1.0 |
| zlib | 1.4 |
| libyaml | 0.2.5 |
| libffi | 3.8.0 |
| GMP | 6.3.0 |
| jemalloc | 5.4.0 |
| libxcrypt | 4.5.2 |

The packages also include `libgcc_s` and `libstdc++` from the build
toolchain. Every ELF file has an `$ORIGIN`-relative `RUNPATH`, so it finds
the bundled libraries wherever the directory is.

### OpenSSL and certificates

The package does not set OpenSSL environment variables. OpenSSL uses
`/etc/ssl` for its default configuration and certificate paths. If your
system uses other paths, set the variables you need before you run Ruby:

- `SSL_CERT_FILE`: path to the system CA bundle.
- `SSL_CERT_DIR`: path to the system CA certificate directory.
- `OPENSSL_CONF`: path to an OpenSSL configuration file.
- `OPENSSL_MODULES`: path to the provider modules. The bundled modules are in
  `<ruby-directory>/lib/ossl-modules`.

For example, on Fedora or RHEL:

```sh
export SSL_CERT_FILE=/etc/pki/tls/certs/ca-bundle.crt
export SSL_CERT_DIR=/etc/pki/tls/certs
export OPENSSL_CONF=/etc/pki/tls/openssl.cnf
export OPENSSL_MODULES=/path/to/ruby/lib/ossl-modules
```

Set only the variables that your system needs.

### Gems with native extensions

`gem install` works for gems with C extensions if the host has a C compiler
(for example `build-essential`). Extensions are compiled with the tier's
`-march` flags from `RbConfig`.

## Build locally

Build on x86_64 Linux. The glibc version of the build host sets the minimum
glibc version of the package.

```sh
sudo apt-get install build-essential curl patchelf pkg-config rustc xz-utils

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

The **Build and release** workflow runs after each push to `main`, every day
at 02:17 UTC, or when you start it from the Actions tab. It reads the latest
two stable versions from the
[ruby/ruby releases](https://github.com/ruby/ruby/releases). CI reuses package
archives that already exist in this repository's releases and builds only the
missing CPU tiers. The build script gets each source URL and SHA256 from Ruby's
official release index.

## License

Ruby and the bundled libraries keep their own licenses: the Ruby License
or BSD-2-Clause for Ruby, Apache-2.0 for OpenSSL, LGPL-3.0-or-later or
GPL-2.0-or-later for GMP, LGPL-2.1-or-later for libxcrypt, BSD-2-Clause for
jemalloc, MIT for libyaml and libffi, and the zlib license for zlib.
