# Betaflight SITL toolchain image.
#
# Native compile only — SITL uses the host `gcc`, not arm-none-eabi.
# Source is mounted at runtime under /src so iterative rebuilds reuse ccache.

FROM ubuntu:24.04

ENV DEBIAN_FRONTEND=noninteractive \
    CCACHE_DIR=/ccache \
    PATH=/usr/lib/ccache:$PATH

RUN apt-get update && apt-get install -y --no-install-recommends \
        build-essential \
        ca-certificates \
        ccache \
        git \
        make \
        python3 \
        ruby \
    && rm -rf /var/lib/apt/lists/*

# Match host UID/GID at build time so files written into the mounted
# source tree don't end up owned by root.
ARG HOST_UID=1000
ARG HOST_GID=1000
RUN if ! getent group ${HOST_GID} >/dev/null; then groupadd -g ${HOST_GID} bfbuild; fi \
 && if ! id -u ${HOST_UID} >/dev/null 2>&1; then \
        useradd -m -u ${HOST_UID} -g ${HOST_GID} -s /bin/bash bfbuild; \
    fi \
 && install -d -o ${HOST_UID} -g ${HOST_GID} /ccache /src

USER ${HOST_UID}:${HOST_GID}
WORKDIR /src

# Default command prints what the image is for; the Makefile overrides this.
CMD ["bash", "-lc", "echo 'Betaflight SITL builder image. Use the host Makefile (build/run/shell).' && echo 'Mount source at /src and run: make TARGET=SITL'"]
