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
- [ ] Docker Desktop with the WSL2 backend
- [ ] Ollama running a local 8B model on the GPU
- [ ] Open WebUI in Docker, connected to Ollama
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

## What I'm Learning

- How WSL2 runs a real Linux kernel in a lightweight VM managed by Windows
- SSH key authentication versus passwords
- Keeping secrets out of public repositories
