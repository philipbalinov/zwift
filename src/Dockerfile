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

SHELL ["/bin/bash", "-o", "pipefail", "-c"]

# As at April 2026 Wine 11.x (devel) includes NTSYNC and WoW64 rewrites
# for significantly improved performance. For a specific version pin add
# WINE_VERSION, make sure to add "=" to the start, comment out for latest:
#    WINE_VERSION="=11.7~trixie-1"
ARG DEBIAN_VERSION
ARG WINE_BRANCH="devel"
ARG WINE_VERSION="=11.7~${DEBIAN_VERSION}-1"
ARG WINETRICKS_VERSION=20260125

# Install prerequisites
# - ca-certificates for wget and curl
# - curl used in zwift authentication script
# - gamemode for freedesktop screensaver inhibit
# - gosu for invoking scripts in entrypoint
# - libegl1 and libgl1 for GL library
# - libvulkan1 for vulkan loader library
# - procps for pgrep
# - wget for downloading winehq key
# - winbind for ntml_auth required by zwift/wine
# - xdg-utils seems to be a dependency of wayland
RUN dpkg --add-architecture i386 \
 && apt-get update \
 && apt-get install --no-install-recommends -y \
        aria2 \
        cabextract \
        ca-certificates \
        curl \
        gamemode \
        gosu \
        innoextract \
        libegl1 \
        libgl1 \
        libvulkan1 \
        procps \
        wget \
        winbind \
        xdg-utils \
        xvfb \
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

# Pre-install Zwift during image build using a virtual display.
# innoextract extracts only the launcher — ZwiftApp.exe is downloaded in the next step.
RUN mkdir -p "/home/user/.wine/drive_c/Program Files (x86)/Zwift" \
 && chown -R user:user /home/user \
 && mkdir -p /run/user/1000 && chown user:user /run/user/1000 \
 && Xvfb :99 -screen 0 1280x1024x24 -nolisten tcp & sleep 2 \
 && gosu user env DISPLAY=:99 XDG_RUNTIME_DIR=/run/user/1000 WINEPREFIX=/home/user/.wine \
        WINEDLLOVERRIDES="mscoree,mshtml=" wineboot -u \
 && gosu user env DISPLAY=:99 XDG_RUNTIME_DIR=/run/user/1000 WINEPREFIX=/home/user/.wine \
        winetricks -q dotnet20 dotnet48 d3dcompiler_47 \
 && aria2c -x 16 -s 16 -d /tmp -o webview2-setup.exe \
        'https://go.microsoft.com/fwlink/p/?LinkId=2124703' \
 && gosu user env DISPLAY=:99 XDG_RUNTIME_DIR=/run/user/1000 WINEPREFIX=/home/user/.wine \
        wine /tmp/webview2-setup.exe /silent /install \
 && gosu user env DISPLAY=:99 XDG_RUNTIME_DIR=/run/user/1000 WINEPREFIX=/home/user/.wine \
        wine reg.exe add 'HKCU\Software\Wine\Drivers' /v Graphics /d x11,wayland \
 && aria2c -x 16 -s 16 -d /tmp -o ZwiftSetup.exe \
        'https://cdn.zwift.com/app/ZwiftSetup.exe' \
 && innoextract --extract --output-dir /tmp/zwift-extract /tmp/ZwiftSetup.exe \
 && cp -r /tmp/zwift-extract/app/. "/home/user/.wine/drive_c/Program Files (x86)/Zwift/" \
 && chown -R user:user "/home/user/.wine/drive_c/Program Files (x86)/Zwift" \
 && gosu user env WINEPREFIX=/home/user/.wine wineserver -k; \
    rm -rf /tmp/zwift-extract /tmp/ZwiftSetup.exe /tmp/webview2-setup.exe \
           /home/user/.cache /tmp/.wine-0

# Run the Zwift launcher to download the full game (ZwiftApp.exe + assets) into the image.
# This layer is cached by Docker/Podman and only re-runs when earlier layers change.
RUN Xvfb :99 -screen 0 1280x1024x24 -nolisten tcp & sleep 2 \
 && zwift_latest=$(wget -qO- 'https://cdn.zwift.com/gameassets/Zwift_Updates_Root/Zwift_ver_cur.xml' \
        | grep -oP 'sversion="\K[^"]+' | cut -f1 -d' ') \
 && [ -n "$zwift_latest" ] \
 && gosu user env DISPLAY=:99 XDG_RUNTIME_DIR=/run/user/1000 WINEPREFIX=/home/user/.wine \
        wine "/home/user/.wine/drive_c/Program Files (x86)/Zwift/ZwiftLauncher.exe" & \
 && for i in $(seq 1 720); do \
        zwift_cur=$(grep -oP 'sversion="\K[^"]+' \
            "/home/user/.wine/drive_c/Program Files (x86)/Zwift/Zwift_ver_cur.xml" \
            2>/dev/null | cut -f1 -d' ') || true; \
        [ "$zwift_cur" = "$zwift_latest" ] && break; \
        sleep 5; \
    done \
 && [ "$zwift_cur" = "$zwift_latest" ] \
 && pkill -f ZwiftApp.exe 2>/dev/null; \
    gosu user env WINEPREFIX=/home/user/.wine wineserver -k || true


# Install fonts for correct Unicode/emoji rendering
RUN gosu user env DISPLAY=:99 XDG_RUNTIME_DIR=/run/user/1000 WINEPREFIX=/home/user/.wine \
    winetricks -q corefonts tahoma unifont dxvk \
&& aria2c -x 16 -s 16 -d /home/user/.wine/drive_c/windows/Fonts \
    'https://github.com/googlefonts/noto-emoji/raw/main/fonts/NotoColorEmoji.ttf' \
&& chown -R user:user /home/user/.wine/drive_c/windows/Fonts

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
