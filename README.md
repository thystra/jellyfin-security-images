# Hardened Jellyfin images

Security-maintained Jellyfin container variants built from pinned upstream images and qualified in CI.

The repository publishes a small set of explicit, versioned images. Each tag identifies the Jellyfin version, ArgentWolf security patch level, and—where applicable—the PostgreSQL client major version. Only the tags listed below are maintained.

## Which image should I use?

For standard Jellyfin 10.11.11 on `linux/amd64` or `linux/arm64`:

```text
ghcr.io/thystra/jellyfin-hardened:10.11.11-awsec1
```

For Jellyfin 10.11.11 with PostgreSQL 17 client tools:

```text
ghcr.io/thystra/jellyfin-pgsql-hardened:10.11.11-awsec2-pg17
```

For Jellyfin 10.11.11 with PostgreSQL 18 client tools:

```text
ghcr.io/thystra/jellyfin-pgsql-hardened:10.11.11-awsec2-pg18
```

All three tags are OCI multi-platform images for `linux/amd64` and `linux/arm64`; Docker or Podman selects the matching platform automatically.

The PostgreSQL variants use the same hardened FFmpeg build as the stock image. Their PostgreSQL client version is selected explicitly so Jellyfin.Pgsql backup and restore subprocesses use the requested `pg_dump`, `pg_restore`, and `psql` major version.

## Security patch levels

### AWSEC2

Jellyfin: 10.11.11  
Jellyfin FFmpeg: 7.1.4-3+awsec1

AWSEC2 is cumulative. It retains the AWSEC1 mitigation for **CVE-2026-8461** and
backports the production changes from upstream Jellyfin PR
[jellyfin/jellyfin#18278](https://github.com/jellyfin/jellyfin/pull/18278), which
shipped in Jellyfin 12.2.

The Jellyfin backport changes local-image timestamp comparisons to tolerate the
precision loss introduced when PostgreSQL stores .NET timestamps at microsecond
precision. This prevents an unchanged local image from being treated as modified
solely because the stored timestamp has lower precision, avoiding the resulting
unnecessary item saves and `ItemUpdated` events during no-change library scans.

The backport is built from the exact Jellyfin v10.11.11 source commit
`1fbd8739292cce610231be93daf43368733edf63`. CI requires the upstream patch to
apply cleanly before building and records the source/backport provenance inside
the image.

### AWSEC1

Jellyfin: 10.11.11  
Jellyfin FFmpeg: 7.1.4-3+awsec1

AWSEC1 mitigates **CVE-2026-8461** by disabling the vulnerable MagicYUV decoder at build time while retaining Jellyfin's expected hardware-acceleration support.

Future security rebuilds will increment the patch level (`awsec2`, `awsec3`, and so on) and document the security changes included in each level. The patch level is cumulative for a given Jellyfin release unless explicitly documented otherwise.

## Upstream bases

The stock image is based on the upstream `ghcr.io/jellyfin/jellyfin` 10.11.11 multi-platform image index pinned by digest. That index resolves to Jellyfin's platform-specific 10.11.11 images for amd64 and arm64.

The PostgreSQL variants are based on the previously qualified `ghcr.io/rogly-net/jellyfin-postgresql` multi-platform image pinned by digest, with the requested PostgreSQL client major selected in the derivative image.

## Architecture policy

AWSEC2 is built and qualified for both `linux/amd64` and `linux/arm64`. Architecture is intentionally not encoded in the image tag; each release tag is a multi-platform OCI image.

The architecture-specific hardened FFmpeg packages are selected during the image build and verified by SHA-256. CI also verifies that the installed Debian package architecture matches the target platform.

There is deliberately no `latest` tag. A moving `latest` tag would imply qualification against Jellyfin releases that this repository has not tested.

## OCI image metadata

Published derivatives override inherited OCI identity labels so consumers can identify this repository as the source of the hardened image. Labels include the source and documentation URLs, image title and description, security-build version, and the exact repository commit used for the build.

## CI qualification

Pull requests build and qualify every image and architecture without publishing. Pushes to `main` publish only after all six variant/platform combinations pass. The publish job then creates multi-platform GHCR images with SBOM and build provenance attestations.

The current CI verifies:

- Jellyfin reports version 10.11.11 on amd64 and arm64;
- the AWSEC2 source marker records the pinned v10.11.11 commit and upstream PR #18278 backport;
- the exact `jellyfin-ffmpeg7` 7.1.4-3+awsec1 package is installed for the target architecture;
- FFmpeg reports Jellyfin 7.1.4;
- the MagicYUV decoder is absent;
- expected hardware-acceleration interfaces remain present for each architecture;
- the arm64 build retains the Rockchip MPP H.264 decoder;
- a real libx264 encode succeeds;
- OCI source, documentation, title, description, version, and revision labels match the expected build metadata;
- PostgreSQL variants select the requested `pg_dump`, `pg_restore`, and `psql` major version;
- PostgreSQL variants complete a real dump-and-restore round trip against a matching PostgreSQL server in CI.

## Source layout

```text
patches/jellyfin-10.11.11-pr18278.patch  upstream PR #18278 backport
images/stock/Dockerfile                 stock Jellyfin 10.11.11 + AWSEC2 fixes
images/pgsql/Dockerfile                 PostgreSQL 17/18 client variants + AWSEC2 fixes
Dockerfile                              compatibility entry point; defaults to PostgreSQL 18
```

## Scope

These images are narrowly maintained derivatives for specifically documented security fixes and upstream bug-fix backports. The project does **not** claim that every CVE reported against every package in the upstream image is fixed.
