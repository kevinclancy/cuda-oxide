# Deploying cuda-oxide on RunPod

`deploy/Dockerfile` builds a self-contained image with the full cuda-oxide
toolchain (CUDA 13.0, LLVM 21, Rust nightly) so it can run as a plain container
on a GPU host such as RunPod. Unlike `.devcontainer/`, it does not rely on the
dev container CLI or VS Code Dev Containers — the Rust toolchain is baked in.

## 1. Build and push to Docker Hub

RunPod GPU hosts are x86_64, so build for `linux/amd64` even from an arm64 Mac.
Make sure Docker Desktop is running first.

```bash
docker login                       # log in to Docker Hub as kevinclancy
docker buildx build \
  --platform linux/amd64 \
  -f deploy/Dockerfile \
  -t kevinclancy/cuda-oxide:latest \
  --push .
```

`--push` uploads straight from buildx (no separate `docker push` needed). The
amd64 build runs under emulation on Apple Silicon, so it will be slow.

To also tag a versioned release, add another `-t`, e.g.
`-t kevinclancy/cuda-oxide:cuda13.0-llvm21`.

## 2. Launch on RunPod

1. Create a GPU pod and set the **Container Image** to
   `docker.io/kevinclancy/cuda-oxide:latest`.
2. Pick a GPU with a driver compatible with CUDA 13.0.
3. **Expose TCP port 22** on the pod (Edit Pod → Expose TCP Ports → add `22`).
   The image runs its own `sshd` as the container's main process, so once the
   pod is "Running" the **Connect → SSH over exposed TCP** panel gives a direct
   `ssh root@<ip> -p <port>` command. Only the developer public key baked into
   the Dockerfile is authorized (key-only login; no passwords).

> Note: RunPod reassigns the public IP and external port on **every pod
> restart**. Re-check the Connect panel and update `~/.ssh/config` after each
> restart. RunPod's web-proxy SSH (`...@ssh.runpod.io`) works for a terminal but
> **cannot be used by VS Code Remote-SSH** — that needs the direct TCP endpoint.

## 3. Edit remotely with VS Code (Remote-SSH)

You do *not* use "Reopen in Container" on RunPod. Instead:

1. Install the **Remote - SSH** VS Code extension locally.
2. Add the pod's direct-TCP connection (from **Connect → SSH over exposed TCP**)
   to your `~/.ssh/config`, e.g.:

   ```
   Host runpod
       HostName <pod-ip>
       Port <external-port>
       User root
       IdentityFile ~/.ssh/id_ed25519
   ```
3. Connect (**Remote-SSH: Connect to Host → runpod**), then **Open Folder** on
   the pod and clone the repo:

   ```bash
   cd /workspaces
   git clone https://github.com/<you>/cuda-oxide.git
   cd cuda-oxide
   cargo oxide doctor
   cargo oxide run vecadd
   ```

4. Install the same extensions you use in the dev container, on the remote:
   `rust-lang.rust-analyzer`, `vadimcn.vscode-lldb`, `tamasfe.even-better-toml`.

The environment variables the build sets (`CUDA_HOME`, `LIBCLANG_PATH`,
`CUDA_OXIDE_LLC`, etc.) match the dev container, so the toolchain behaves the
same way.

## Keeping in sync with the dev container

The Rust channel/components mirror `rust-toolchain.toml` and
`.devcontainer/devcontainer.json`. If you bump the Rust nightly or the CUDA/LLVM
versions there, update the `ARG`/`FROM` lines in `deploy/Dockerfile` too.
