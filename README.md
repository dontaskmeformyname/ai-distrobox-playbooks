# Local AI – Ansible Playbooks (Docker, GPU‑agnostic)

This repository now contains a **stand‑alone Ansible role** `hermes_container` that builds a Docker image with:

* `hermes‑agent` (latest release)
* optional **Ollama** binary (so you keep your existing Ollama‑cloud provider)
* a **llama.cpp** server (lightweight, GPU‑friendly GGUF models)
* a **persistent Docker volume** `hermes-data` that stores `~/.hermes` (config, memories, downloaded models, API keys, etc.)

# 2. Bild-KI: ComfyUI + PyTorch-ROCm im selben Container einrichten
ansible-playbook 02_image_ai_setup.yml

# 3. Multi-User-AI-Stack (unabhängig von 01/02, Quadlet statt Distrobox)
ansible-playbook 03_ai_stack.yml --ask-become-pass
```

## Running on a Proxmox host (192.168.178.3)

| Playbook | Inhalt |
|---|---|
| `01_text_ai_setup.yml` | Distrobox Container `ai-box`, ROCm, Ollama, Open WebUI (Port 8080) |
| `02_image_ai_setup.yml` | ComfyUI, PyTorch-ROCm, Modell-Verzeichnisse, Start-Skript (Port 8188) |
| `03_ai_stack.yml` | Distro-agnostischer Multi-User-AI-Stack (Podman Quadlet): Ollama/vLLM, Hermes Agent Server, Open WebUI, Authelia SSO |

## Nach der Installation (Playbooks 01/02)

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

Die Distrobox-Playbooks (01/02) können oben in der `vars`-Sektion angepasst werden:

- `container_name`: Name des Distrobox-Containers (Standard: `ai-box`)
- `hsa_override`: GPU-Ziel-Architektur (RX 6900 XT = `10.3.0`)
- `ollama_keep_alive`: `0` = VRAM sofort freigeben (empfohlen bei gleichzeitigem Betrieb)
- `download_example_model`: `true` um SDXL automatisch herunterzuladen

## 03_ai_stack.yml – Multi-User-AI-Stack (Quadlet)

Unabhängig von den Distrobox-Playbooks (01/02): richtet einen dienst-orientierten
Multi-User-AI-Stack als rootless Podman-Quadlet-Deployment ein. Alle Dienste laufen
als systemd-User-Units des Users `aisvc` in einem gemeinsamen Pod (`ai.pod`) und
erreichen sich intern über `127.0.0.1`.

Distro-agnostisch: Ubuntu/Debian, RHEL/Fedora/Rocky, openSUSE/SLE, Arch/CachyOS.
GPU-Erkennung erfolgt automatisch (NVIDIA via CDI, AMD via `/dev/kfd` + `/dev/dri`).

| Dienst | Port | Zweck |
|---|---|---|
| Ollama API | 11434 | LLM-Inference mit GPU-Passthrough (`inference_engine: vllm` als Alternative, nur NVIDIA) |
| Open WebUI | 8080 | Chat-Frontend, Login via OIDC (Authelia) |
| Hermes Gateway | 9119 | Hermes Agent Server, Remote-Connect (Desktop: Settings → Gateway → Remote gateway) |
| Authelia | 9091 | OIDC-IdP für zentrales User-Management |

### Voraussetzungen

- Podman >= 4.4 (Quadlet-Support, wird automatisch geprüft)
- Ubuntu: `universe`-Repo aktiv; SLE: container-tools-Modul aktiviert

### Vor dem Ausführen anpassen

- `oidc_client_secret`: `CHANGE_ME_RANDOM_SECRET` ersetzen (Ansible Vault oder `--extra-vars` empfohlen, nicht committen)
- `ai_models`: zu ziehende Ollama-Modelle
- `inference_engine`: `ollama` (Standard) oder `vllm` (nur NVIDIA sinnvoll, AMD-ROCm-Support von vLLM ist experimentell)
- Authelia-Bootstrap: Passworthashes (`authelia crypto hash generate argon2`), Session-Secret und OIDC-Client in `configuration.yml` ergänzen (Redirect-URI: `http://<webui-host>:8080/oauth/oidc/callback`)

### Betrieb

```bash
sudo -u aisvc systemctl --user status ollama hermes open-webui authelia
sudo -u aisvc journalctl --user -u ollama -f
sudo -u aisvc podman stats
```

### Sicherheitshinweise

- Port 9119 (Hermes Gateway) und 9091 (Authelia) nicht öffentlich exponieren – Hermes hat kein OIDC, nur eigene Credentials. Zugriff über WireGuard/Tailscale oder Reverse Proxy.
- Ollama (11434) hat keine Auth – bei LAN-Betrieb PublishPort auf `127.0.0.1:11434:11434` binden.
