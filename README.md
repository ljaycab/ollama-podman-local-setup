# Ollama + Podman Local Chat Setup Guide

A complete step-by-step tutorial for running Ollama with Podman on Linux Mint, with common errors and solutions.

**Difficulty Level:** Intermediate | **Time:** ~30 minutes | **Prerequisites:** Linux Mint/Ubuntu

---

## 📋 Quick Navigation

- [Prerequisites](#prerequisites)
- [Part 1: Install Ollama](#part-1-install-ollama)
- [Part 2: Create Podman Sandbox](#part-2-create-podman-sandbox)
- [Part 3: Download a Model](#part-3-download-a-model)
- [Part 4: Create and Run Chat Script](#part-4-create-and-run-chat-script)
- [⚠️ Common Errors](./docs/TROUBLESHOOTING.md)
- [Daily Usage](./docs/DAILY_USAGE.md)
- [Cleanup](./docs/CLEANUP.md)

---

## Prerequisites

- Linux Mint (or Ubuntu-based system)
- `podman` installed: `sudo apt install podman`
- `curl` installed: `sudo apt install curl`
- ~8 GB free disk space (for Mistral model)
- Internet connection

**Verify Prerequisites:**
```bash
podman --version
curl --version
df -h /home  # Check free space
