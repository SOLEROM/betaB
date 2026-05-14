# Betaflight cross-compile environment
# Ubuntu 22.04 + ARM GNU Toolchain 13.3.Rel1 (matches mk/tools.mk in betaflight)

FROM ubuntu:22.04

ARG TARGETARCH
ARG ARM_GCC_VERSION=13.3.rel1
ARG USER_UID=1000
ARG USER_GID=1000

ENV DEBIAN_FRONTEND=noninteractive \
    LANG=C.UTF-8 \
    LC_ALL=C.UTF-8

# Build deps + things Betaflight Makefile/scripts call
RUN apt-get update && apt-get install -y --no-install-recommends \
        build-essential \
        ca-certificates \
        curl \
        git \
        make \
        python3 \
        python3-pip \
        ruby \
        unzip \
        xxd \
        xz-utils \
        sudo \
    && rm -rf /var/lib/apt/lists/*

# Install ARM GNU Toolchain — pick tarball by container arch
RUN set -eux; \
    case "${TARGETARCH:-$(dpkg --print-architecture)}" in \
        amd64) ARM_ARCH=x86_64 ;; \
        arm64) ARM_ARCH=aarch64 ;; \
        *) echo "Unsupported arch: ${TARGETARCH}"; exit 1 ;; \
    esac; \
    URL="https://developer.arm.com/-/media/Files/downloads/gnu/${ARM_GCC_VERSION}/binrel/arm-gnu-toolchain-${ARM_GCC_VERSION}-${ARM_ARCH}-arm-none-eabi.tar.xz"; \
    mkdir -p /opt/arm-toolchain; \
    curl -fsSL "$URL" | tar -xJ -C /opt/arm-toolchain --strip-components=1; \
    /opt/arm-toolchain/bin/arm-none-eabi-gcc --version

ENV PATH="/opt/arm-toolchain/bin:${PATH}"

# Non-root user matching host UID/GID so volume writes stay user-owned
RUN groupadd -g "${USER_GID}" dev 2>/dev/null || true; \
    useradd -m -u "${USER_UID}" -g "${USER_GID}" -s /bin/bash dev; \
    echo "dev ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/dev

USER dev
WORKDIR /work

CMD ["/bin/bash"]
