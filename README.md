# Local AI – Ansible Playbooks (Docker, GPU‑agnostic)

This repository now contains a **stand‑alone Ansible role** `hermes_container` that builds a Docker image with:

* `hermes‑agent` (latest release)
* optional **Ollama** binary (so you keep your existing Ollama‑cloud provider)
* a **llama.cpp** server (lightweight, GPU‑friendly GGUF models)
* a **persistent Docker volume** `hermes-data` that stores `~/.hermes` (config, memories, downloaded models, API keys, etc.)

The role can be run on a **local workstation** (Docker on Ubuntu 22.04) **or** on a remote **Proxmox** host (via SSH).  All heavy files (models, secrets) live in the Docker volume, so you can delete/re‑create the container without losing state.

---

## New usage: Deploy Hermes Agent with a local LLM backend

1. **Clone the repo (SSH – your SSH‑agent is already forwarded):**
   ```bash
   git clone git@github.com:alex/ai-distrobox-playbooks.git
   cd ai-distrobox-playbooks
   ```
2. **Provide your Ollama‑cloud API key** (never commit it):
   ```bash
   echo "hermes_api_key_ollama_cloud: \"YOUR_OLLAMA_CLOUD_KEY\"" > roles/hermes_container/vars/secret.yml
   ```
   The file is already listed in `.gitignore`.
3. **(Optional) Choose which backend you want:** edit `inventory/group_vars/all.yml` and set `hermes_backend` to either:
   * `ollama` – uses the full Ollama binary (recommended if you already have many models there).
   * `llama_cpp` – uses the tiny `llama-server` with the two GGUF models that fit a 16 GB RX 6900 XT and provide the required **≥ 64 K** context.
4. **Run the playbook locally:**
   ```bash
   ansible-playbook -i inventory site.yml --ask-become-pass
   ```
   This will:
   * build the Docker image `hermes-agent:ubuntu22`
   * create the Docker volume `hermes-data`
   * copy a minimal `config.yaml` (points Hermes at the local backend) and a `.env` with your cloud key into the volume
   * pull the two GPU‑friendly GGUF models (`yarn‑mistral:7b‑64k‑q5_0` and `qwen3.5:9b‑64k‑q8_0`) via `ollama pull`
   * start a container exposing the Hermes daemon on port **5000**
5. **Verify the service:**
   ```bash
   curl -s http://127.0.0.1:5000/model | jq .providers
   # should list two groups: "Local GGUF Server" and "Ollama Cloud"
   hermes chat -q "Say exactly: OK" -p custom -t safe --max-turns 1
   ```
6. **Persisted data:** All Hermes data lives in the Docker volume `hermes-data`.  You can inspect it with:
   ```bash
   docker run --rm -v hermes-data:/hermes busybox ls -l /hermes
   ```
   Deleting the container (`docker rm -f hermes-agent`) does **not** delete the volume.

---

## Running on a Proxmox host (192.168.178.3)

1. Ensure the target host can run Docker (install Docker Engine on Proxmox if not present).
2. The same inventory file already contains a `[proxmox]` group pointing at `192.168.178.3`.
3. Execute the playbook against that host:
   ```bash
   ansible-playbook -i inventory site.yml -l proxmox --ask-become-pass
   ```
   The playbook will SSH into the Proxmox host, build the exact same image there, create the volume, and start the container.  The Hermes API will then be reachable at `http://192.168.178.3:5000`.

---

## Quick reference cheat‑sheet for the GPU‑friendly models

| Model (Ollama name)                | GGUF file (after `ollama export`) | Quantisation | Approx. VRAM @ 64 K context | Reason for inclusion |
|------------------------------------|----------------------------------|--------------|-----------------------------|----------------------|
| `yarn-mistral:7b-64k-q5_0`          | `yarn-mistral_7b_64k_q5_0.gguf` | Q5_0 (~4 GB) | ≈ 9 GB (weights + KV)      | 7 B model, fits comfortably into 16 GB VRAM, already ships a 64 K context window.
| `qwen3.5:9b-64k-q8_0`              | `qwen3.5_9b_64k_q8_0.gguf`       | Q8_0 (~6 GB) | ≈ 12 GB (weights + KV)      | Slightly larger model, still under the 16 GB ceiling; excellent for coding tasks.

Both models meet **Hermes Agent's hard minimum of 64 K tokens** and are small enough for the RX 6900 XT.

---

## Cleaning up / rebuilding

```bash
# Stop and remove the container (does NOT delete the persistent volume)
docker rm -f hermes-agent

# If you also want to wipe the persisted Hermes data:
#   (caution – you lose memories, installed skills, etc.)
# docker volume rm hermes-data

# Re‑run the playbook to rebuild everything from scratch
ansible-playbook -i inventory site.yml --ask-become-pass
```

---

## What changed in this repository

* Added `roles/hermes_container/` (Dockerfile, Ansible role, templates, defaults, vars).
* New `inventory/hosts.ini` with a `[local]` and `[proxmox]` entry.
* New `inventory/group_vars/all.yml` (sets `hermes_backend` and Python path).
* Updated `README.md` with full instructions for both local and Proxmox deployments.
* Added `.gitignore` entry for `roles/hermes_container/vars/secret.yml` (never commit the API key).

You can now treat this repo as a **complete, reproducible bootstrap** for a Hermes Agent instance that works on your GPU‑limited hardware **and** keeps access to the Ollama cloud provider.
