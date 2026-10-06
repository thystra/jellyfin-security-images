# syntax=docker/dockerfile:1

# Build the two Jellyfin server assemblies changed by upstream PR #18278.
# Source is pinned to the exact v10.11.11 commit.
FROM --platform=$BUILDPLATFORM mcr.microsoft.com/dotnet/sdk:9.0-bookworm-slim AS jellyfin-pr18278

ARG TARGETARCH
ARG JELLYFIN_SOURCE_COMMIT="1fbd8739292cce610231be93daf43368733edf63"

RUN set -eux; \
    apt-get update; \
    DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends ca-certificates git; \
    rm -rf /var/lib/apt/lists/*; \
    git init /src/jellyfin; \
    cd /src/jellyfin; \
    git remote add origin https://github.com/jellyfin/jellyfin.git; \
    git fetch --depth 1 origin "${JELLYFIN_SOURCE_COMMIT}"; \
    git checkout --detach FETCH_HEAD; \
    test "$(git rev-parse HEAD)" = "${JELLYFIN_SOURCE_COMMIT}"

COPY patches/jellyfin-10.11.11-pr18278.patch /tmp/jellyfin-pr18278.patch

RUN set -eux; \
    cd /src/jellyfin; \
    git apply --check /tmp/jellyfin-pr18278.patch; \
    git apply /tmp/jellyfin-pr18278.patch; \
    case "${TARGETARCH}" in \
      amd64) DOTNET_ARCH="x64" ;; \
      arm64) DOTNET_ARCH="arm64" ;; \
      *) echo "Unsupported architecture: ${TARGETARCH}" >&2; exit 1 ;; \
    esac; \
    dotnet publish Jellyfin.Server \
      --arch "${DOTNET_ARCH}" \
      --configuration Release \
      --output /out \
      --self-contained \
      -p:DebugSymbols=false \
      -p:DebugType=none; \
    test -f /out/MediaBrowser.Controller.dll; \
    test -f /out/MediaBrowser.Providers.dll; \
    printf 'Jellyfin source commit: %s\nBackport: jellyfin/jellyfin#18278 (merged as 2a59779ae98d96be782078ddf414040d931831ea)\n' \
      "${JELLYFIN_SOURCE_COMMIT}" > /out/AWSEC2-SOURCE.txt

# Jellyfin 10.11.11 multi-platform image index, pinned by digest.
FROM ghcr.io/jellyfin/jellyfin@sha256:45f648c382a0c8b552582fcea40e95cb17c5d475473a891cba0eb7523fb92112

COPY --from=jellyfin-pr18278 /out/MediaBrowser.Controller.dll /jellyfin/MediaBrowser.Controller.dll
COPY --from=jellyfin-pr18278 /out/MediaBrowser.Providers.dll /jellyfin/MediaBrowser.Providers.dll
COPY --from=jellyfin-pr18278 /out/AWSEC2-SOURCE.txt /usr/share/jellyfin-awsec/AWSEC2-SOURCE.txt

USER root

ARG TARGETARCH
ARG FFMPEG_SHA256_AMD64="625065a539e3209717a977e9aede107bf61c4e0fc16ba78b54dea12f8ac0329b"
ARG FFMPEG_SHA256_ARM64="7d64e9d56b7679a0c928306574382e4b97fe4b861c7ab44b03dceda0cb23341a"
ARG GIT_REVISION="unknown"

RUN set -eux; \
    case "${TARGETARCH}" in \
      amd64) FFMPEG_SHA256="${FFMPEG_SHA256_AMD64}" ;; \
      arm64) FFMPEG_SHA256="${FFMPEG_SHA256_ARM64}" ;; \
      *) echo "Unsupported architecture: ${TARGETARCH}" >&2; exit 1 ;; \
    esac; \
    FFMPEG_URL="https://github.com/thystra/jellyfin-ffmpeg/releases/download/v7.1.4-3-awsec1/jellyfin-ffmpeg7_7.1.4-3%2Bawsec1-trixie_${TARGETARCH}.deb"; \
    apt-get update; \
    apt-get install -y --no-install-recommends ca-certificates curl; \
    curl -fL "${FFMPEG_URL}" -o /tmp/jellyfin-ffmpeg.deb; \
    echo "${FFMPEG_SHA256}  /tmp/jellyfin-ffmpeg.deb" | sha256sum -c -; \
    apt-get install -y --no-install-recommends /tmp/jellyfin-ffmpeg.deb; \
    rm -f /tmp/jellyfin-ffmpeg.deb; \
    apt-get clean; \
    rm -rf /var/lib/apt/lists/*

LABEL org.opencontainers.image.source="https://github.com/thystra/jellyfin-security-images" \
      org.opencontainers.image.url="https://github.com/thystra/jellyfin-security-images" \
      org.opencontainers.image.documentation="https://github.com/thystra/jellyfin-security-images/blob/main/README.md" \
      org.opencontainers.image.title="Hardened Jellyfin" \
      org.opencontainers.image.description="Jellyfin 10.11.11 with the AWSEC1 FFmpeg mitigation for CVE-2026-8461 and upstream PR #18278 image-date backport" \
      org.opencontainers.image.version="10.11.11-awsec2" \
      org.opencontainers.image.revision="${GIT_REVISION}"
