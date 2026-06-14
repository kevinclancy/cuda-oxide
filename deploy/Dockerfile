# Copyright (c) 2024-2026 NVIDIA CORPORATION. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
#
# Self-contained image for RunPod (and other plain `docker run` deployments).
#
# Unlike .devcontainer/Dockerfile, this image bakes in the Rust nightly
# toolchain that the dev container otherwise layers on via dev container
# features. The result is a single image that needs no devcontainer CLI to
# build and runs standalone on a GPU host. Edit the project on the pod with
# VS Code Remote-SSH; clone the repo into /workspaces at runtime.
#
# Build for the architecture RunPod runs (x86_64), even from an arm64 Mac:
#
#   docker buildx build --platform linux/amd64 \
#     -f deploy/Dockerfile -t kevinclancy/cuda-oxide:latest --push .

FROM nvidia/cuda:13.0.0-devel-ubuntu24.04

ARG DEBIAN_FRONTEND=noninteractive

# Keep these in sync with rust-toolchain.toml.
ARG RUST_CHANNEL=nightly-2026-04-03
ARG RUST_COMPONENTS=rust-src,rustc-dev,rust-analyzer,rustfmt,clippy,llvm-tools

ENV CUDA_HOME=/usr/local/cuda \
    CUDA_PATH=/usr/local/cuda \
    CUDA_TOOLKIT_PATH=/usr/local/cuda \
    CUDA_OXIDE_LLC=/usr/bin/llc-21 \
    LIBCLANG_PATH=/usr/lib/llvm-21/lib \
    LLVM_CONFIG_PATH=/usr/bin/llvm-config-21 \
    RUSTUP_HOME=/usr/local/rustup \
    CARGO_HOME=/usr/local/cargo \
    PATH=/usr/local/cargo/bin:/usr/lib/llvm-21/bin:/usr/local/cuda/bin:${PATH}

SHELL ["/bin/bash", "-o", "pipefail", "-c"]

RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        ca-certificates \
        curl \
        g++ \
        gcc \
        git \
        gnupg \
        libc6-dev \
        make \
        openssh-server \
        pkg-config \
        python3 \
        python3-pip \
        xz-utils \
    && curl -fsSL https://apt.llvm.org/llvm-snapshot.gpg.key \
        | gpg --dearmor -o /usr/share/keyrings/apt.llvm.org.gpg \
    && echo "deb [signed-by=/usr/share/keyrings/apt.llvm.org.gpg] https://apt.llvm.org/noble/ llvm-toolchain-noble-21 main" \
        > /etc/apt/sources.list.d/llvm-toolchain-noble-21.list \
    && apt-get update \
    && apt-get install -y --no-install-recommends \
        clang-21 \
        libclang-common-21-dev \
        lld-21 \
        llvm-21 \
        llvm-21-dev \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# Install the Rust nightly toolchain that the dev container provides via the
# devcontainers/features/rust feature.
RUN curl --proto '=https' --tlsv1.2 -fsSL https://sh.rustup.rs \
        | sh -s -- -y \
            --no-modify-path \
            --profile minimal \
            --default-toolchain "${RUST_CHANNEL}" \
            --component "${RUST_COMPONENTS}" \
    && chmod -R a+w "${RUSTUP_HOME}" "${CARGO_HOME}" \
    && rustc --version \
    && cargo --version

# Docker `ENV` (above) only applies to the container's main process, not to
# interactive SSH/login shells, which re-derive PATH from profile files. Without
# this, `cargo`/`llc`/`nvcc` are "not found" over SSH and rust-analyzer (run by
# the VS Code remote server) can't locate the toolchain. Mirror the ENV into a
# profile.d script so every login shell — and Remote-SSH — sees it.
RUN cat > /etc/profile.d/cuda-oxide.sh <<'EOF'
export CUDA_HOME=/usr/local/cuda
export CUDA_PATH=/usr/local/cuda
export CUDA_TOOLKIT_PATH=/usr/local/cuda
export CUDA_OXIDE_LLC=/usr/bin/llc-21
export LIBCLANG_PATH=/usr/lib/llvm-21/lib
export LLVM_CONFIG_PATH=/usr/bin/llvm-config-21
export RUSTUP_HOME=/usr/local/rustup
export CARGO_HOME=/usr/local/cargo
export PATH=/usr/local/cargo/bin:/usr/lib/llvm-21/bin:/usr/local/cuda/bin:$PATH
EOF

# Configure sshd so the container is reachable over direct TCP (required for VS
# Code Remote-SSH; RunPod's web proxy exec's a shell in without sshd, but
# Remote-SSH needs a real sshd listening on 22). openssh-server is installed
# above. Root login by key only — no passwords.
RUN mkdir -p /run/sshd /root/.ssh \
    && chmod 700 /root/.ssh \
    && sed -i \
        -e 's/^#\?PermitRootLogin.*/PermitRootLogin prohibit-password/' \
        -e 's/^#\?PasswordAuthentication.*/PasswordAuthentication no/' \
        -e 's/^#\?PubkeyAuthentication.*/PubkeyAuthentication yes/' \
        /etc/ssh/sshd_config

# Authorize the developer's public key. This is a *public* key, so committing it
# is safe. Replace it (or add lines) to grant other users access.
RUN echo 'ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIMKi49uV/TnNxWuHjNN+b3yNlqdVkh63apwOCpVh1mFV kclanc@gmail.com' \
        > /root/.ssh/authorized_keys \
    && chmod 600 /root/.ssh/authorized_keys

WORKDIR /workspaces

EXPOSE 22

# Run sshd in the foreground as PID 1. This both serves SSH *and* keeps the
# container alive on hosts that start it detached (e.g. RunPod) — a bare `bash`
# with no TTY would read EOF and exit, stopping the container. `-D` keeps sshd
# in the foreground; `-e` logs to stderr so `docker logs` shows auth attempts.
CMD ["/usr/sbin/sshd", "-D", "-e"]
