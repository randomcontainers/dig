# dig

Container images with the DNS client tools of [ISC BIND 9](https://www.isc.org/bind/): `dig`, `host`, `nslookup`, `delv`, `nsupdate` and `mdig`, compiled from ISC's signed release tarball on Ubuntu or Alpine. The images follow the BIND 9.20 series and are rebuilt when ISC publishes a 9.20 release and when the base image changes, for `linux/amd64` and `linux/arm64`.

This is an unofficial build, not affiliated with or endorsed by ISC. Report problems with the image in this repository and problems with the tools themselves [upstream](https://gitlab.isc.org/isc-projects/bind9).

## Quick start

```sh
docker run --rm ghcr.io/randomcontainers/dig example.org
```

Print only the answer, from a resolver of your choice:

```sh
docker run --rm ghcr.io/randomcontainers/dig @9.9.9.9 +short example.org AAAA
```

Show the DNSSEC keys of a zone with their key tags:

```sh
docker run --rm ghcr.io/randomcontainers/dig +dnssec +multi isc.org DNSKEY
```

The entrypoint runs `dig` under `tini`. Without `@server`, dig asks the resolvers in the container's `/etc/resolv.conf`. Docker fills that file from the host's settings, or with its own DNS server at 127.0.0.11 on a user-defined network. Run without arguments, the image prints dig's help. The [BIND 9 manual](https://bind9.readthedocs.io/en/bind-9.20/manpages.html) describes every option of the tools.

## The other tools

`host`, `nslookup`, `delv`, `nsupdate` and `mdig` are in `/usr/local/bin` next to `dig`. `mdig` is a dig that sends all of its queries before it reads the answers. Run any of them with `--entrypoint`:

```sh
docker run --rm --entrypoint host ghcr.io/randomcontainers/dig -t MX example.org
```

`delv` checks the DNSSEC chain of trust of an answer itself, starting from the root key built into it, and prints `; fully validated` when it holds:

```sh
docker run --rm --entrypoint delv ghcr.io/randomcontainers/dig isc.org
```

`nslookup` and `nsupdate` read commands interactively when they get a terminal, with line editing:

```sh
docker run --rm -it --entrypoint nslookup ghcr.io/randomcontainers/dig
```

## Encrypted DNS

`+https` sends the query over DNS over HTTPS and `+tls` over DNS over TLS. dig does not check the server's certificate unless you pass `+tls-ca`, which verifies it against the CA certificates in the image and the name given with `@`:

```sh
docker run --rm ghcr.io/randomcontainers/dig @dns.quad9.net +https +tls-ca example.org
```

## Dynamic updates

`nsupdate` reads update commands from a file and signs them with a TSIG key from a key file, in the `key "<name>" { algorithm ...; secret "..."; };` format that BIND's `tsig-keygen` writes. With `tsig.key` and `update.txt` in the current directory:

```
server ns1.example.com
zone example.com
update delete www.example.com A
update add www.example.com 300 A 192.0.2.10
send
```

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work:ro" \
  --entrypoint nsupdate ghcr.io/randomcontainers/dig -k tsig.key update.txt
```

The directory is mounted read-only, since nsupdate only reads the two files.

## Debugging DNS in containers and clusters

On a user-defined Docker network, Docker's DNS server answers for the names of the other containers:

```sh
docker run --rm --network mynet ghcr.io/randomcontainers/dig +short web
```

In Kubernetes, a one-off pod uses the cluster's DNS settings, so `+search` resolves a service name the way the cluster's workloads do:

```sh
kubectl run dns-debug --rm -it --restart=Never \
  --image=ghcr.io/randomcontainers/dig -- +search +short kubernetes.default
```

## What is in the image

- `dig`, `host`, `nslookup`, `delv`, `nsupdate` and `mdig` in `/usr/local/bin`.
- The four BIND libraries they load, `libisc`, `libdns`, `libisccfg` and `libns` of the same release, in `/usr/local/lib`.

They are built with DNS over HTTPS (nghttp2) and DNS over TLS (OpenSSL), internationalized domain names through libidn2, GSS-TSIG for `nsupdate -g` through MIT Kerberos, and line editing through libedit. For GSS-TSIG, mount a Kerberos ticket cache and `/etc/krb5.conf` into the container; the image has no `kinit`.

Not included: `named` and the other server and DNSSEC signing tools (`rndc`, `named-checkzone`, `dnssec-keygen` and so on), dnstap, the jemalloc allocator, the man pages and the headers. The configure flags and the versions of the libraries the tools were built against are in `/usr/local/share/randomcontainers/dig/buildinfo`.

## Default or slim

dig's default image adds no other tools, so `latest` and `slim` are the same image: the BIND client tools and the libraries they link against. Use `latest` to run them and the `slim` tags as a base for your own image.

## Tags

`<version>` is a BIND release such as `9.20.29`. `<minor>` and `<major>` are its shorter forms, `9.20` and `9`, and follow the newest release in that series. Each row lists the default tag and its `slim` twin, which point to the same image.

| Tags | Base |
|---|---|
| `latest`, `slim` | Ubuntu |
| `<version>`, `<version>-slim` | Ubuntu |
| `<minor>`, `<minor>-slim`, `<major>`, `<major>-slim` | Ubuntu |
| `ubuntu`, `slim-ubuntu` | Ubuntu |
| `<version>-ubuntu`, `<version>-slim-ubuntu` | Ubuntu |
| `<minor>-ubuntu`, `<minor>-slim-ubuntu`, `<major>-ubuntu`, `<major>-slim-ubuntu` | Ubuntu |
| `<version>-ubuntu26.04`, `<version>-slim-ubuntu26.04` | Ubuntu 26.04 |
| `alpine`, `slim-alpine` | Alpine |
| `<version>-alpine`, `<version>-slim-alpine` | Alpine |
| `<minor>-alpine`, `<minor>-slim-alpine`, `<major>-alpine`, `<major>-slim-alpine` | Alpine |
| `<version>-alpine3.24`, `<version>-slim-alpine3.24` | Alpine 3.24 |

The images are currently built on Ubuntu 26.04 and Alpine 3.24. Tags without a distro version move to the next distro release when the project does; tags ending in `ubuntu26.04` or `alpine3.24` stay on that release and are no longer rebuilt once the project moves to the next one. Every tag of the current BIND version, including the exact version, is rebuilt in place (see [Updates](#updates)), so pin a digest when you need the same bytes every time.

## Platforms

`linux/amd64` and `linux/arm64`, for both Ubuntu and Alpine. Both are compiled natively on GitHub-hosted runners, without emulation.

## Files and permissions

The working directory is `/work`. The image runs as UID 1000, and any other UID works too: `HOME` is then `/`, and caches go to `/cache`, which anyone can write to. How to get output files owned by you depends on how you run containers:

| Runtime | Flag |
|---|---|
| Docker on Linux (rootful), GitHub Actions | `--user "$(id -u):$(id -g)"` |
| Rootless Podman | `--userns=keep-id` |
| Rootless Docker | `--user 0:0` (root in the container is your user on the host) |
| Docker Desktop on macOS or Windows | none, file ownership is mapped for you |

The tools need no capabilities: queries go out from unprivileged ports.

## Extending the slim image

Use a `slim` tag as the base for your own image. `slim`, `slim-ubuntu` and `slim-alpine` move to each new BIND release and are rebuilt when the base image changes. The packages the tools need are listed in `/usr/local/share/randomcontainers/dig/runtime-deps`. Switch to root to install more, then back:

```dockerfile
FROM ghcr.io/randomcontainers/dig:slim-ubuntu@sha256:...
USER root
RUN apt-get update \
 && apt-get install -y --no-install-recommends curl \
 && rm -rf /var/lib/apt/lists/*
USER 1000:1000
```

On Alpine, start from `slim-alpine` and use `apk add --no-cache curl`. The entrypoint is `["tini", "--", "dig"]`; set your own `ENTRYPOINT` if your image runs something else. To pick up new BIND releases and base image fixes, let Dependabot or Renovate update the digest in your `FROM` line.

## Verifying

Each image has a build provenance attestation from this repository's GitHub Actions run, signed by the shared build workflow in `randomcontainers/ci`:

```sh
gh attestation verify oci://ghcr.io/randomcontainers/dig:latest \
  --repo randomcontainers/dig --signer-repo randomcontainers/ci
```

Each platform image also carries an SPDX SBOM that lists every distro package with its version:

```sh
docker buildx imagetools inspect ghcr.io/randomcontainers/dig:latest --format '{{ json .SBOM }}'
```

Before compiling, the build checks the tarball against the SHA-256 recorded in `package.yml` and its signature against ISC's code-signing keys in `keys/isc-keyblock.gpg`. ISC publishes these keys and their fingerprints at [isc.org/pgpkey](https://www.isc.org/pgpkey/), and the Dockerfiles list the fingerprints too.

## Updates

The project checks the `v9.20.<patch>` tags of [isc-projects/bind9](https://github.com/isc-projects/bind9) every 15 minutes. A release is picked up once its tag is 24 hours old and its tarball and signature are on downloads.isc.org, which is usually a few days after ISC tags it. The new version and the tarball's SHA-256 are then committed to `package.yml` and the images are rebuilt. A release signed with a key that is not in `keys/isc-keyblock.gpg` fails to build until the key is added. Only the newest release is built; tags of older versions stay as they were last built.

Only 9.20 releases are followed, the series ISC currently maintains as its extended support version. Moving the images to a later stable series is a change in this repository, not an automatic update.

The images of the current version are also rebuilt when the Ubuntu or Alpine base image changes and at least every 7 days, so distro security fixes, including those for OpenSSL, libuv and nghttp2, reach the current tags.

## Building

```sh
docker build -f Dockerfile.ubuntu --target slim \
  --build-arg VERSION=<version> \
  --build-arg SOURCE_SHA256=<sha256 from package.yml> \
  -t dig:local .
```

Use `Dockerfile.alpine` for the Alpine image. `--build-arg JOBS=<n>` limits the number of parallel compile jobs. The build compiles all of BIND, including `named`, and keeps only the client tools.

## Licenses

BIND 9 is licensed under the Mozilla Public License 2.0 (MPL-2.0). Some of the code compiled into the tools and libraries is under other permissive terms: ISC, two- and three-clause BSD, MIT, and CC0 for SipHash. The image's license label is therefore `MPL-2.0 AND ISC AND BSD-2-Clause AND BSD-3-Clause AND MIT AND CC0-1.0`. The Apache-2.0 code in BIND's tarball is part of dnstap, the DNSRPS interface and the tests, none of which this build compiles. `/usr/local/share/randomcontainers/dig/licenses/` has BIND's `LICENSE` and `COPYRIGHT`, and `NOTICES` with the notices of compiled code that `COPYRIGHT` does not cover: SipHash, picohttpparser, the C-RW-WP reader-writer lock, `strnstr` and the NLnet Labs duration code. The Ubuntu and Alpine packages in the image keep their own licenses.

The source code for each image:

- BIND: every version has a GitHub release in this repository, named `v<version>`, with the exact `bind-<version>.tar.xz` that was compiled and its signature. The build applies no patches. `/usr/local/share/randomcontainers/dig/source` lists that release and the downloads.isc.org URLs.
- Build scripts: this repository at the commit in the image's `org.opencontainers.image.revision` label. The Dockerfiles hold every configure flag.
- Ubuntu packages: the source packages on [Launchpad](https://launchpad.net/ubuntu) for the versions listed in the SBOM. `apt-get source <package>=<version>` fetches a version that is still in the Ubuntu archive.
- Alpine packages: Alpine has no source packages. For the versions listed in the SBOM, the source is the APKBUILD and patches in [aports](https://gitlab.alpinelinux.org/alpine/aports/-/tree/3.24-stable), branch `3.24-stable`, and the archives on [distfiles.alpinelinux.org](https://distfiles.alpinelinux.org/distfiles/v3.24/).

The files in this repository are available under the MIT license, see [LICENSE](LICENSE).

## Requesting a tool

To suggest another tool, use the [Request a tool](https://github.com/randomcontainers/.github/issues/new?template=tool-request.yml) form.
