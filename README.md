# Local AI in Distrobox – Ansible Playbooks
## Hardware-Voraussetzungen
- AMD Radeon RX 6900 XT (RDNA2 / gfx1030)
- 32 GB RAM
- Linux-Host mit Distrobox & Podman installiert

## Reihenfolge

```bash
# 1. Text-KI: Container erstellen, ROCm + Ollama + Open WebUI einrichten
ansible-playbook 01_text_ai_setup.yml --ask-become-pass

# 2. Bild-KI: ComfyUI + PyTorch-ROCm im selben Container einrichten
ansible-playbook 02_image_ai_setup.yml

# 3. Multi-User-AI-Stack (unabhängig von 01/02, Quadlet statt Distrobox)
ansible-playbook 03_ai_stack.yml --ask-become-pass
```

## Was wird eingerichtet?

| Playbook | Inhalt |
|---|---|
| `01_text_ai_setup.yml` | Distrobox Container `ai-box`, ROCm, Ollama, Open WebUI (Port 8080) |
| `02_image_ai_setup.yml` | ComfyUI, PyTorch-ROCm, Modell-Verzeichnisse, Start-Skript (Port 8188) |
| `03_ai_stack.yml` | Distro-agnostischer Multi-User-AI-Stack (Podman Quadlet): Ollama/vLLM, Hermes Agent Server, Open WebUI, Authelia SSO |

## Nach der Installation (Playbooks 01/02)

| Dienst | URL |
|---|---|
| Open WebUI (Chat) | http://localhost:8080 |
| ComfyUI (Bilder) | http://localhost:8188 |
| Ollama API | http://localhost:11434 |

## Modelle laden (Beispiele)

```bash
# Text-Modelle (im Container)
distrobox enter ai-box -- ollama pull llama3.2
distrobox enter ai-box -- ollama pull mistral
distrobox enter ai-box -- ollama pull deepseek-r1:14b

# Bild-Modell herunterladen (SDXL ~6 GB)
# In 02_image_ai_setup.yml: download_example_model: true setzen
# und Playbook erneut ausführen
```

## Wichtige Variablen

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
