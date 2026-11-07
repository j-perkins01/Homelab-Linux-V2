# Home Lab V2: WSL2, Containers and Local AI

A hands-on lab on my Windows 11 PC for building Linux, container and AI skills: a Linux environment in WSL2, container tooling, and a self-hosted AI stack running on my own GPU. It also prepares for a cloud project on AWS.

## Lab Environment

| Component | Details |
|---|---|
| Host | Windows 11 Home |
| CPU | AMD Ryzen 7 5700 (8 cores / 16 threads) |
| RAM | 32 GB DDR4 |
| GPU | NVIDIA RTX 5060 (8 GB VRAM) |
| Linux | Ubuntu 24.04 LTS on WSL2 |

## Progress

- [x] Enable WSL2 and install Ubuntu 24.04
- [x] Update Ubuntu packages
- [x] Configure Git and SSH key authentication to GitHub
- [x] Docker Desktop with the WSL2 backend
- [x] Ollama running a local 8B model on the GPU
- [x] Open WebUI in Docker, connected to Ollama
- [ ] Uptime Kuma monitoring the services

## Setup Log

### 1. WSL2 and Ubuntu

```powershell
wsl --install            # enables WSL and Virtual Machine Platform, then restart
wsl --install -d Ubuntu-24.04
wsl -l -v                # confirm VERSION 2
```

**Issue:** After `wsl --install` and a restart, `wsl --status` reported *"no installed distributions"*. The Windows features were enabled but the distro had not been installed.
**Fix:** Installed the distro explicitly with `wsl --install -d Ubuntu-24.04`.

Then updated packages:

```bash
sudo apt update && sudo apt upgrade -y
```

### 2. Git and GitHub over SSH

```bash
ssh-keygen -t ed25519 -C "github"
cat ~/.ssh/id_ed25519.pub        # added to GitHub > Settings > SSH and GPG keys
ssh -T git@github.com            # "You have successfully authenticated"

git config --global user.name "JTP"
git config --global user.email "<id>+<username>@users.noreply.github.com"
git config --global init.defaultBranch main
```

I used GitHub's noreply address to keep my personal email out of public commit history. A `.gitignore` blocks keys, `.env` files, Terraform state and model files from being committed.

### 3. Docker Desktop with WSL2

Installed Docker Desktop on Windows, then enabled integration for the Ubuntu distro:
**Settings > Resources > WSL integration > Ubuntu-24.04**.

```powershell
winget install --id Docker.DockerDesktop -e
```

Verified from inside Ubuntu:

```bash
docker --version             # Docker 29.9
docker compose version       # Compose v5.6
docker run --rm hello-world  # "Hello from Docker!"
```

Docker runs its engine inside WSL2, so containers get the full CPU (16 threads) and about half the system RAM by default.

### 4. Ollama: local AI on the GPU

Installed Ollama natively on Windows (simplest path to the NVIDIA GPU) and pulled an 8B model:

```powershell
winget install --id Ollama.Ollama -e
ollama pull llama3.1:8b      # 4.9 GB download
ollama run llama3.1:8b
```

Ollama serves a local API on `http://localhost:11434`, which other tools (like Open WebUI) can call.

**Verifying it uses the GPU, not the CPU:**

```text
> ollama ps
NAME          SIZE     PROCESSOR   CONTEXT
llama3.1:8b   5.3 GB   100% GPU    4096

> nvidia-smi --query-gpu=memory.used,memory.total,utilization.gpu --format=csv
7311 MiB, 8151 MiB, 95 %
```

Result: about **30 tokens/second** on the RTX 5060, fully offloaded to the GPU. The model plus context uses about 7.3 of 8 GB VRAM, so 8B is the practical size limit for this card.

Test prompt sent to the API:

```powershell
Invoke-RestMethod http://localhost:11434/api/generate -Method Post `
  -ContentType application/json `
  -Body (@{model="llama3.1:8b"; prompt="What is AWS Lambda?"; stream=$false} | ConvertTo-Json)
```

### 5. Open WebUI in Docker

A ChatGPT-style web interface for the local model, defined in [`open-webui/docker-compose.yml`](open-webui/docker-compose.yml).

```bash
cd open-webui
docker compose up -d
curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:3000/health   # 200
```

Then open `http://localhost:3000` and create the first account (it becomes the admin; accounts are stored locally in the container volume).

**Design decisions:**

- **Ollama stays on Windows, the UI runs in a container.** Windows 11 Home has no GPU passthrough to VMs, so running Ollama natively is the reliable way to use the GPU. The container reaches it at `http://host.docker.internal:11434`.
- **Bound to `127.0.0.1` only.** The port mapping `127.0.0.1:3000:8080` means only this PC can open the UI, not other devices on the network.
- **Named volume for data.** Chats and settings live in the `open-webui-data` volume, so they survive container updates and restarts.

Verified the container can reach Ollama:

```bash
docker exec open-webui curl -s http://host.docker.internal:11434/api/tags   # lists llama3.1:8b
```

## What I'm Learning

- How WSL2 runs a real Linux kernel in a lightweight VM managed by Windows
- SSH key authentication versus passwords
- Keeping secrets out of public repositories
- What containers are and why they are used instead of installing apps directly
- How model size relates to GPU memory (VRAM), and how to check where a model is running
- How a container talks to a service on the host (`host.docker.internal`), and why binding ports to localhost matters
