ARG DEBIAN_VERSION=trixie

FROM rust:1.90-slim AS build-runfromprocess

# Install prerequisites
# - mingw to cross-compile for windows
# - git to fetch runfromprocess source code
# - rust cross-compiler for windows
RUN apt-get update \
 && apt-get install --no-install-recommends -y \
        g++-mingw-w64-x86-64 \
        git \
 && rm -rf /var/lib/apt/lists/* \
 && rustup target add x86_64-pc-windows-gnu

# Build runfromprocess
WORKDIR /usr/src
RUN git clone https://github.com/quietvoid/runfromprocess-rs . \
 && git checkout d8754035b1b8a38f96ec6806a772a944ef5396c0 \
 && cargo build --target x86_64-pc-windows-gnu --release

FROM debian:${DEBIAN_VERSION}-slim AS wine-base

# As at April 2026 Wine 11.x (devel) includes NTSYNC and WoW64 rewrites
# for significantly improved performance. For a specific version pin add
# WINE_VERSION, make sure to add "=" to the start, comment out for latest:
#    WINE_VERSION="=11.7~trixie-1"
ARG DEBIAN_VERSION
ARG WINE_BRANCH="devel"
ARG WINE_VERSION="=11.7~${DEBIAN_VERSION}-1"
ARG WINETRICKS_VERSION=20260125

# Install prerequisites
# - aria2 for parallel downloads in update/install scripts
# - cabextract and innoextract for winetricks and Zwift installer extraction
# - fonts-liberation, fonts-noto-color-emoji, fonts-unifont for Unicode/emoji rendering in Wine
# - ca-certificates for wget and curl
# - curl used in zwift authentication script
# - gamemode for freedesktop screensaver inhibit
# - gosu for invoking scripts in entrypoint
# - gstreamer1.0-libav and gstreamer1.0-plugins-ugly for H.264 encoding (activity videos)
# - libegl1 and libgl1 for GL library
# - libvulkan1 for vulkan loader library
# - procps for pgrep
# - wget for downloading winehq key
# - winbind for ntml_auth required by zwift/wine
# - xdg-utils seems to be a dependency of wayland
RUN dpkg --add-architecture i386 \
 && sed -i 's/^Components: main$/Components: main contrib non-free non-free-firmware/' /etc/apt/sources.list.d/debian.sources \
 && apt-get update \
 && apt-get install --no-install-recommends -y \
        aria2 \
        cabextract \
        ca-certificates \
        curl \
        gamemode \
        gosu \
        fonts-liberation \
        fonts-noto-color-emoji \
        fonts-unifont \
        gstreamer1.0-libav \
        gstreamer1.0-plugins-ugly \
        innoextract \
        libegl1 \
        libgl1 \
        libvulkan1 \
        procps \
        wget \
        winbind \
        xdg-utils \
 && rm -rf /var/lib/apt/lists/*

# Install wine and winetricks (including recommends, which appear to be required)
# hadolint ignore=DL3015
RUN wget -qO /etc/apt/trusted.gpg.d/winehq.asc https://dl.winehq.org/wine-builds/winehq.key \
 && echo "deb https://dl.winehq.org/wine-builds/debian/ ${DEBIAN_VERSION} main" > /etc/apt/sources.list.d/winehq.list \
 && apt-get update \
 && apt-get install -y \
        wine-${WINE_BRANCH}${WINE_VERSION} \
        wine-${WINE_BRANCH}-amd64${WINE_VERSION} \
        wine-${WINE_BRANCH}-i386${WINE_VERSION} \
        winehq-${WINE_BRANCH}${WINE_VERSION} \
 && rm -rf /var/lib/apt/lists/* \
 && wget -qO /usr/local/bin/winetricks https://raw.githubusercontent.com/Winetricks/winetricks/${WINETRICKS_VERSION}/src/winetricks \
 && chmod +x /usr/local/bin/winetricks

# Create user and make nvidia libraries discoverable
RUN adduser --disabled-password --gecos '' user \
 && echo "/usr/local/nvidia/lib" >> /etc/ld.so.conf.d/nvidia.conf \
 && echo "/usr/local/nvidia/lib64" >> /etc/ld.so.conf.d/nvidia.conf

# Required for non-glvnd setups
ENV LD_LIBRARY_PATH=/usr/lib/x86_64-linux-gnu:/usr/lib/i386-linux-gnu:/usr/local/nvidia/lib:/usr/local/nvidia/lib64

# Configure audio driver
COPY pulse-client.conf /etc/pulse/client.conf

FROM wine-base

ENV NVIDIA_VISIBLE_DEVICES=all
ENV NVIDIA_DRIVER_CAPABILITIES=all
ENV WINEDEBUG=fixme-all

LABEL org.opencontainers.image.authors="Kim Eik <kim@heldig.org>"
LABEL org.opencontainers.image.title="netbrain/zwift"
LABEL org.opencontainers.image.description="Easily zwift on linux"
LABEL org.opencontainers.image.url="https://github.com/netbrain/zwift"

COPY --chmod=755 entrypoint.sh /bin/entrypoint
COPY --chmod=755 update_zwift.sh /bin/update_zwift.sh
COPY --chmod=755 run_zwift.sh /bin/run_zwift.sh
COPY --chmod=755 zwift-auth.sh /bin/zwift-auth
COPY --chmod=755 --from=build-runfromprocess /usr/src/target/x86_64-pc-windows-gnu/release/runfromprocess-rs.exe /bin/runfromprocess-rs.exe

ENTRYPOINT ["entrypoint"]
