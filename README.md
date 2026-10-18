# postgresql-client

Container images with the client programs of [PostgreSQL](https://www.postgresql.org/): `psql`, `pg_dump`, `pg_restore`, `pg_basebackup` and the others, with the `libpq` library they share. They are compiled from the PostgreSQL source release against the libraries of Ubuntu or Alpine, for `linux/amd64` and `linux/arm64`, and the images are rebuilt when PostgreSQL publishes a release and when the base image changes. The server is not included.

This is an unofficial build, not affiliated with or endorsed by the PostgreSQL Global Development Group. Report problems with the image in this repository, and bugs in PostgreSQL as its [bug reporting guidelines](https://www.postgresql.org/docs/current/bug-reporting.html) describe.

## Quick start

Open `psql` on a database:

```sh
docker run --rm -it ghcr.io/randomcontainers/postgresql-client -h db.example.com -U app shop
```

The entrypoint runs `psql` under `tini` in `/work`, so file names are relative to the directory you mount. The other programs run with `--entrypoint`. A few more commands, with your server in place of `db.example.com`:

```sh
# Run a query and print the result as CSV, with the password from your shell's PGPASSWORD
docker run --rm -e PGPASSWORD ghcr.io/randomcontainers/postgresql-client \
  -h db.example.com -U app -d shop --csv -c 'select * from orders limit 10'

# Run a SQL file and stop at the first error
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" -e PGPASSWORD \
  ghcr.io/randomcontainers/postgresql-client \
  -h db.example.com -U app -d shop --set ON_ERROR_STOP=1 -f migrate.sql

# Check whether a server accepts connections
docker run --rm --entrypoint pg_isready ghcr.io/randomcontainers/postgresql-client \
  -h db.example.com
```

`pg_isready` tries to connect once. It exits with 0 when the server accepts connections, 1 when it rejects them (during startup, for example), 2 when there is no response and 3 when it made no attempt because of bad parameters. `-t` limits how many seconds that one attempt may take (3 by default). To wait for a server, for example in CI, run it in a loop:

```sh
until docker run --rm --entrypoint pg_isready ghcr.io/randomcontainers/postgresql-client \
    -h db.example.com -t 5; do sleep 2; done
```

The [client applications reference](https://www.postgresql.org/docs/current/reference-client.html) describes every program and option.

## Connecting

A server on another host is reached by name or address. For a server on the Docker host itself, use `--network host -h 127.0.0.1` on Linux, or `-h host.docker.internal` with Docker Desktop. To connect through the server's Unix socket, mount its directory and pass it as the host; Debian and Ubuntu servers put the socket in `/var/run/postgresql`:

```sh
docker run --rm -it --user "$(id -u):$(id -g)" \
  -v /var/run/postgresql:/var/run/postgresql \
  ghcr.io/randomcontainers/postgresql-client -h /var/run/postgresql -U "$(id -un)" -d shop
```

With `peer` authentication the server looks up the UID of the client process on the host and, unless a user name map says otherwise, accepts only the role of the same name. `--user "$(id -u):$(id -g)"` runs the programs with your UID, and `-U "$(id -un)"` asks for your role. Without `-U`, libpq takes the user name from the image's `/etc/passwd`, which knows nothing about your account on the host. Without `-h`, the programs look for the socket in `/tmp`, where it is only found if you mount it there.

Passwords:

- `-e PGPASSWORD` passes the variable from your shell. Avoid `-e PGPASSWORD=secret` on the command line, which ends up in your shell history and in `ps` output.
- `HOME` is `/cache`, so libpq reads the password file from `/cache/.pgpass`. Mount yours with `--user "$(id -u):$(id -g)" -v "$HOME/.pgpass:/cache/.pgpass:ro"`: libpq ignores the file unless its mode is 0600 or stricter, and with that mode only your UID can read it. `-e PGPASSFILE=/work/pgpass` points to another file.
- Connection service files work the same way: `/cache/.pg_service.conf`, or `PGSERVICEFILE`.

TLS uses OpenSSL. To check a server certificate signed by a public CA against the CA certificates of the image, pass `sslrootcert=system`, which also turns on `sslmode=verify-full`:

```sh
docker run --rm -it ghcr.io/randomcontainers/postgresql-client \
  'host=db.example.com user=app dbname=shop sslrootcert=system'
```

For a private CA, mount its certificate and pass `sslmode=verify-full sslrootcert=/work/ca.crt`, or mount it at `/cache/.postgresql/root.crt`, where libpq looks by default. A client certificate and key go in `sslcert` and `sslkey`, or in `/cache/.postgresql/postgresql.crt` and `postgresql.key`.

Kerberos (GSSAPI) uses MIT Kerberos. The image has no `kinit`, so get a ticket on the host, then mount your `/etc/krb5.conf` and the credential cache and point `KRB5CCNAME` at it. This works with a file cache, such as `/tmp/krb5cc_<uid>`, the default on Debian and Ubuntu:

```sh
docker run --rm -it --user "$(id -u):$(id -g)" \
  -v /etc/krb5.conf:/etc/krb5.conf:ro -v "/tmp/krb5cc_$(id -u):/cache/krb5cc" \
  -e KRB5CCNAME=FILE:/cache/krb5cc \
  ghcr.io/randomcontainers/postgresql-client -h db.example.com -U app shop
```

The cache is mounted writable because the Kerberos library adds the ticket for the server to it. `klist` on the host shows which kind of cache you have. A `KCM:` cache (the default on Fedora) or a `KEYRING:` cache cannot be mounted; `kinit -c FILE:/tmp/krb5cc_pg` gets a ticket into a file, which you then mount in place of `/tmp/krb5cc_<uid>`.

## Backups

`pg_dump` and `pg_restore` work with files in the mounted directory. Dump a database to a compressed archive, then restore it into an existing database:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" -e PGPASSWORD --entrypoint pg_dump \
  ghcr.io/randomcontainers/postgresql-client -h db.example.com -U app -Fc -Z zstd -f shop.dump shop

docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" -e PGPASSWORD --entrypoint pg_restore \
  ghcr.io/randomcontainers/postgresql-client -h db.example.com -U app -d shop --no-owner shop.dump
```

`-Z` takes `gzip`, `lz4` or `zstd`, optionally with a level such as `zstd:9`. `-Fd -j 4` writes a directory archive with four parallel jobs, and `pg_restore -j 4` restores it in parallel. A plain SQL dump needs no mount when you redirect it on the host:

```sh
docker run --rm -e PGPASSWORD --entrypoint pg_dump ghcr.io/randomcontainers/postgresql-client \
  -h db.example.com -U app shop > shop.sql
```

`pg_basebackup` copies a whole server over the replication protocol, as a superuser or a role with the `REPLICATION` attribute, and `pg_verifybackup` checks the copy against its manifest:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" -e PGPASSWORD --entrypoint pg_basebackup \
  ghcr.io/randomcontainers/postgresql-client -h db.example.com -U replicator -D base -P
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" --entrypoint pg_verifybackup \
  ghcr.io/randomcontainers/postgresql-client base
```

`pg_verifybackup` runs `pg_waldump` to check the WAL of the backup, which is why `pg_waldump` is in the image.

## Server versions

The image has the client programs of the newest PostgreSQL major version only (see [Tags](#tags)). They work with servers of older versions, with these limits from the PostgreSQL documentation:

- `pg_dump` dumps servers back to 9.2 and refuses to dump a server of a newer major version than its own.
- A dump is meant to be loaded into a server of the same or a newer version. Loading it into an older major version can fail, even when the dump was taken from a server of that version: `pg_dump` 18 writes `SET transaction_timeout = 0;`, which servers before 17 reject. To move data to an older server, edit the dump or use the `pg_dump` of that server's version.
- Plain-text dumps contain `\restrict` and `\unrestrict` lines. `psql` understands them from 13.22, 14.19, 15.14, 16.10 and 17.6 on, and in every release of 18 and later; older `psql` releases and other SQL clients reject them. Load such a dump with the `psql` of this image, or dump with `-Fc` and restore the archive with `pg_restore`.
- `psql` works best with servers of the same or an older major version. Its `\d` commands work with servers back to 9.2, and backslash commands may fail against a newer server.
- `pg_basebackup` works with servers of the same or an older major version, down to 9.1.

## What is in the image

- `psql`, `pg_dump`, `pg_dumpall`, `pg_restore`, `pg_basebackup`, `pg_receivewal`, `pg_recvlogical`, `pg_verifybackup`, `pg_combinebackup`, `pg_waldump`, `pg_amcheck`, `pgbench`, `pg_isready`, `createdb`, `dropdb`, `createuser`, `dropuser`, `clusterdb`, `reindexdb` and `vacuumdb` in `/usr/local/bin`.
- `libpq.so.5` in `/usr/local/lib`, with TLS from OpenSSL and GSSAPI from MIT Kerberos.
- `pg_dump`, `pg_restore` and the backup programs compress with zlib (gzip), LZ4 and Zstandard, and `psql` edits lines with libedit. These libraries, OpenSSL and Kerberos come from the distro.

Not included: the server and its tools (`postgres`, `initdb`, `pg_ctl`, `pg_upgrade`, `pg_rewind`, `pg_createsubscriber` and the others), `pg_config`, `ecpg`, the C headers, the static library and the pkg-config file, the man pages, translations of the messages, GNU readline, ICU, LDAP lookup of connection parameters, and OAuth authentication, which needs libcurl. The configure flags are in `/usr/local/share/randomcontainers/postgresql-client/buildinfo`.

`psql` shows long results through `more`. Its `\e` command needs an editor: the Alpine image has BusyBox `vi`, the Ubuntu image has none.

## Default or slim

The postgresql-client default image adds no other tools, so `latest` and `slim` are the same image, with the contents listed above. Use `latest` to run it and the `slim` tags as a base for your own image. The default image of [pgcli](https://github.com/randomcontainers/pgcli) includes this build of the client programs.

## Tags

`<version>` is a PostgreSQL release such as `18.6`. `<major>` is its first number, `18`, and follows the newest release with that number. Each row lists the default tag and its `slim` twin, which point to the same image.

| Tags | Base |
|---|---|
| `latest`, `slim` | Ubuntu |
| `<version>`, `<version>-slim` | Ubuntu |
| `<major>`, `<major>-slim` | Ubuntu |
| `ubuntu`, `slim-ubuntu` | Ubuntu |
| `<version>-ubuntu`, `<version>-slim-ubuntu` | Ubuntu |
| `<major>-ubuntu`, `<major>-slim-ubuntu` | Ubuntu |
| `<version>-ubuntu26.04`, `<version>-slim-ubuntu26.04` | Ubuntu 26.04 |
| `alpine`, `slim-alpine` | Alpine |
| `<version>-alpine`, `<version>-slim-alpine` | Alpine |
| `<major>-alpine`, `<major>-slim-alpine` | Alpine |
| `<version>-alpine3.24`, `<version>-slim-alpine3.24` | Alpine 3.24 |

The images are currently built on Ubuntu 26.04 and Alpine 3.24. Tags without a distro version move to the next distro release when the project does; tags ending in `ubuntu26.04` or `alpine3.24` stay on that release and are no longer rebuilt once the project moves to the next one. Every tag of the current PostgreSQL version, including the exact version, is rebuilt in place (see [Updates](#updates)), so pin a digest when you need the same bytes every time.

Only the newest major version is built. When a new one is released, `latest`, the distro tags and the `slim` tags move to it, and the `<major>` tags of the previous version stay on its last release built here. Later releases of that version are not built, so an image on such a tag gets no more fixes, from PostgreSQL or from the distro. Releases of older major versions are not built at all.

## Platforms

`linux/amd64` and `linux/arm64`, for both Ubuntu and Alpine. Both are compiled natively on GitHub-hosted runners, without emulation.

## Files and permissions

The working directory is `/work`. The image runs as UID 1000, and any other UID works too: `HOME` is `/cache` for every user, and `/cache` can be written by anyone. With another UID, pass the database user with `-U` or `PGUSER`: libpq takes the default user name from the image's `/etc/passwd`, and for a UID without an entry there it stops with `local user with ID <n> does not exist`. How to get output files owned by you depends on how you run containers:

| Runtime | Flag |
|---|---|
| Docker on Linux (rootful), GitHub Actions | `--user "$(id -u):$(id -g)"` |
| Rootless Podman | `--userns=keep-id` |
| Rootless Docker | `--user 0:0` (root in the container is your user on the host) |
| Docker Desktop on macOS or Windows | none, file ownership is mapped for you |

Under `/cache`, `psql` keeps its history in `.psql_history` and runs `.psqlrc` at startup (`-X` skips it), and libpq reads `.pgpass`, `.pg_service.conf` and the certificates in `.postgresql/`. Mount a volume on `/cache` to keep them between runs, for example `-v psql-home:/cache`. `/usr/local/etc/postgresql/psqlrc` and `/usr/local/etc/postgresql/pg_service.conf` are read for every user when they exist.

## Extending the slim image

Use a `slim` tag as the base for your own image. `slim`, `slim-ubuntu` and `slim-alpine` move to each new PostgreSQL release, including new major versions, and are rebuilt when the base image changes. Switch to root to install more, then back:

```dockerfile
FROM ghcr.io/randomcontainers/postgresql-client:slim-ubuntu@sha256:...
USER root
RUN apt-get update \
 && apt-get install -y --no-install-recommends less \
 && rm -rf /var/lib/apt/lists/*
ENV PSQL_PAGER="less -S"
USER 1000:1000
```

`less -S` lets you scroll wide results sideways. On Alpine, start from `slim-alpine` and use `apk add --no-cache less`. The entrypoint is `["tini", "--", "psql"]`; set your own `ENTRYPOINT` if your image runs something else. To pick up new PostgreSQL releases and base image fixes, let Dependabot or Renovate update the digest in your `FROM` line.

Ubuntu's `libpq5` and Alpine's `libpq` packages have the same soname as `/usr/local/lib/libpq.so.5`. If a package you add depends on them, that package may load either copy; the client programs here keep loading the one in `/usr/local/lib`.

Everything the image adds is under `/usr/local`. `/usr/local/share/randomcontainers/postgresql-client/` holds the version, the source URL, the build options, the license files and `runtime-deps`, the list of distro packages the programs need at run time.

## Verifying

Each image has a build provenance attestation from this repository's GitHub Actions run, signed by the shared build workflow in `randomcontainers/ci`:

```sh
gh attestation verify oci://ghcr.io/randomcontainers/postgresql-client:latest \
  --repo randomcontainers/postgresql-client --signer-repo randomcontainers/ci
```

Each platform image also carries an SPDX SBOM that lists every distro package with its version:

```sh
docker buildx imagetools inspect ghcr.io/randomcontainers/postgresql-client:latest --format '{{ json .SBOM }}'
```

PostgreSQL publishes a `.sha256` file next to each source tarball but no signatures. Before compiling, the build checks the tarball against the SHA-256 recorded in `package.yml`. The SBOM covers distro packages only. PostgreSQL is compiled from source. Its version is in `/usr/local/share/randomcontainers/postgresql-client/version`, and the tarball's SHA-256 is in `buildinfo` next to it.

## Updates

The project checks the [source directory](https://ftp.postgresql.org/pub/source/) on postgresql.org every 15 minutes. Each release gets a directory there, `v<version>/`, on the day it is announced; betas and release candidates are skipped. A release is picked up once its directory is at least 48 hours old. Its tarball is checked against the `postgresql-<version>.tar.bz2.sha256` file next to it, the new version and the tarball's SHA-256 are committed to `package.yml`, and the images are rebuilt. PostgreSQL publishes a new major version about once a year and minor releases at least every three months. Only the newest release is built; tags of older versions stay as they were last built.

The images of the current version are also rebuilt when the Ubuntu or Alpine base image changes and at least every 7 days, so distro security fixes, including those for OpenSSL and Kerberos, reach the current tags.

## Building

```sh
docker build -f Dockerfile.ubuntu --target slim \
  --build-arg VERSION=<version> \
  --build-arg SOURCE_SHA256=<sha256 from package.yml> \
  -t postgresql-client:local .
```

Use `Dockerfile.alpine` for the Alpine image. `--build-arg JOBS=<n>` limits the number of parallel compile jobs.

## Licenses

PostgreSQL is licensed under the [PostgreSQL License](https://www.postgresql.org/about/licence/), a permissive license similar to the BSD and MIT licenses. Some code compiled into the programs is under other licenses:

- BSD-3-Clause: `snprintf.c`, `pgmkdirp.c` and `bsearch_arg.c` from `src/port`, and the sort routine in `src/include/lib/sort_template.h`, under notices of the Regents of the University of California, and for `snprintf.c` also of Eric P. Allman.
- ISC: `inet_net_ntop.c` from the Internet Systems Consortium, and `timingsafe_bcmp.c` from Damien Miller, which replaces a function that glibc and musl lack.
- MIT: the fasthash functions by Zilong Tan in `src/include/common/hashfn_unstable.h`, which `pg_dumpall`, `pg_verifybackup` and `pg_combinebackup` use for hash tables, and CRC-32C code generated with [fast-crc32](https://github.com/corsix/fast-crc32), which `pg_waldump`, `pg_verifybackup` and `pg_combinebackup` use for checksums. Which file of `src/port` holds that code depends on the platform: `pg_crc32c_sse42.c` on amd64 and, from PostgreSQL 19, also `pg_crc32c_armv8.c` on arm64. The generated code has no copyright notice of its own, so the images whose build compiles it include the MIT license of fast-crc32.
- Unicode-3.0: the tables generated from the Unicode Character Database, which libpq and `psql` use to prepare SCRAM passwords and to work out how wide characters are on screen. The source tarball has no copy of the Unicode License v3, so the text comes from this repository.
- The archive code of `pg_dump` and `pg_restore` carries a notice from Philip Warner that grants use "in any way so long as this notice is not removed". It has no SPDX identifier, so the image's license label names it `LicenseRef-pg-dump`.
- Public domain: Markus Kuhn's `wcwidth` code in `src/common/wchar.c`, which libpq and `psql` use to work out how wide characters are on screen, and Bob Jenkins' hash function in `src/common/hashfn.c`, which `pg_dump` uses for hash tables. The license label names them `LicenseRef-Public-Domain`.

`/usr/local/share/randomcontainers/postgresql-client/licenses/` holds PostgreSQL's `COPYRIGHT` file and these notices, cut from the source files as `LICENSE.snprintf`, `LICENSE.qsort`, `LICENSE.fasthash`, `LICENSE.pg_dump` and so on, with `LICENSE.fast-crc32` (where the fast-crc32 code is compiled) and `Unicode-3.0` from this repository. The image's license label is `PostgreSQL AND BSD-3-Clause AND ISC AND MIT AND Unicode-3.0 AND LicenseRef-pg-dump AND LicenseRef-Public-Domain`. The parser and scanner of `pgbench` and the scanners of `psql` are generated during the build by the distro's Bison and flex, and Bison's license exception allows its output to be used under any terms. `psql` uses libedit, so GNU readline and its GPL license are not part of the image. The Ubuntu and Alpine packages in the image keep their own licenses; the SBOM lists them.

PostgreSQL and Postgres are registered trademarks of the PostgreSQL Community Association of Canada. The files in this repository are available under the MIT license, see [LICENSE](LICENSE), except the license texts in `licenses/`, which keep their own terms.

## Requesting a tool

To suggest another tool, use the [Request a tool](https://github.com/randomcontainers/.github/issues/new?template=tool-request.yml) form.
