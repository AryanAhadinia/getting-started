# CUDA/PyTorch base image that matches EPFL GPUs.
# 25.11 = CUDA 13.0 (native on the RCP nodes' R580 driver), PyTorch 2.10, kernels for
# sm_75, sm_80, sm_86, sm_90, sm_100, sm_120: A100, H100, H200 and B300 (sm_103 runs the
# sm_100 kernels). NGC images from 25.01 on have no V100 (sm_70) kernels, so V100 jobs
# keep using the previous image, mlo-base:uv1 (built from 24.02).
ARG BASE_IMAGE=nvcr.io/nvidia/pytorch:25.11-py3
FROM --platform=linux/amd64 ${BASE_IMAGE}
LABEL maintainer="Alexander Haegele <alexander.hagele@epfl.ch>"

# Allow non-interactive apt installs (no tz prompts)
ARG DEBIAN_FRONTEND=noninteractive
ARG LDAP_USERNAME=cluster
ARG LDAP_UID=1000
ARG LDAP_GROUPNAME=cluster
ARG LDAP_GID=1000

ENV LANG=en_US.UTF-8 \
    LC_ALL=en_US.UTF-8 \
    LANGUAGE=en_US:en \
    SHELL=/bin/zsh \
    TF_FORCE_GPU_ALLOW_GROWTH=true

# Basic tooling for interactive workflows plus sudo for entrypoint tricks
RUN apt-get update && apt-get install -y --no-install-recommends \
    ca-certificates \
    curl \
    git \
    gnupg \
    htop \
    nvtop \
    jq \
    less \
    libssl-dev \
    locales \
    nano \
    openssh-client \
    pkg-config \
    python3-full \
    rsync \
    screen \
    sudo \
    tmux \
    unzip \
    vim \
    wget \
    zsh \
    && rm -rf /var/lib/apt/lists/*

# Generate UTF-8 locale for Python + shells
RUN locale-gen en_US.UTF-8

# Ubuntu 24.04-based NGC images (25.x+) ship an 'ubuntu' user and group at 1000,
# which collides with the placeholder account below
RUN if id -u ubuntu >/dev/null 2>&1; then userdel -r ubuntu; fi

# Placeholder LDAP-like account; entrypoint will later mutate UID/GID
RUN groupadd -g ${LDAP_GID} ${LDAP_GROUPNAME} && \
    useradd -m -s /bin/zsh -g ${LDAP_GROUPNAME} -u ${LDAP_UID} ${LDAP_USERNAME}

# Seed account that already owns /mloscratch; entrypoint impersonates it
RUN useradd -m -s /bin/zsh -g ${LDAP_GROUPNAME} -u 177449 mljaggi-admin

# Pre-install Oh My Zsh once (later copied to scratch-backed homes)
RUN sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)" "" --unattended && \
    chsh -s /bin/zsh root

# Install uv once at build time; per-user caches live on scratch
ENV UV_INSTALL_DIR=/usr/local/bin
RUN curl -LsSf https://astral.sh/uv/install.sh | sh

# Ship entrypoint + cached shell assets inside the image
RUN mkdir -p /docker
COPY entrypoint.sh /docker/entrypoint.sh
RUN chmod +x /docker/entrypoint.sh

# Copy root’s Oh My Zsh artifacts so entrypoint can clone them cheaply
RUN cp -r /root/.oh-my-zsh /docker/.oh-my-zsh && \
    cp /root/.zshrc /docker/.zshrc && \
    echo "umask 007" >> /docker/.zshrc

# Set root password only because some infra still demands it
RUN echo 'root:root' | chpasswd

WORKDIR /mloscratch/homes/
ENTRYPOINT ["/docker/entrypoint.sh"]
CMD ["/bin/zsh"]