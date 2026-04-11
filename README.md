# Local AI – Ansible Playbooks (Podman, GPU-agnostisch)

Ansible-Playbooks zur automatischen Einrichtung einer lokalen KI-Umgebung mit **Podman**-Containern.
Funktioniert sowohl auf einem **Headless-Server** als auch auf einem **Desktop-System**.

## Unterstützte GPUs

| GPU-Hersteller | Beschleunigung | Erkennungsmethode |
|---|---|---|
| NVIDIA | CUDA | `nvidia-smi` |
| AMD | ROCm | `rocminfo` |
| Intel | oneAPI / Level Zero | `/dev/dri` + Vendor-ID |
| Kein GPU | CPU-only Fallback | automatisch |

Die GPU-Erkennung erfolgt **vollautomatisch** zu Beginn des Playbooks.
Die benötigten Treiber/Bibliotheken werden entsprechend im Container installiert.

## System-Voraussetzungen

- Linux-Host (x86_64)
- Podman installiert (`podman --version`)
- Ansible installiert (`ansible --version`)
- Für AMD: `rocminfo` verfügbar, `/dev/kfd` vorhanden
- Für NVIDIA: `nvidia-smi` verfügbar, NVIDIA Container Toolkit installiert
- Für Intel: `/dev/dri/renderD128` vorhanden

## Installation

```bash
# 1. Text-KI: Podman Container, GPU-Treiber, Ollama, Open WebUI
ansible-playbook 01_text_ai_setup.yml --ask-become-pass

# 2. Bild-KI: ComfyUI + PyTorch (GPU-angepasst)
ansible-playbook 02_image_ai_setup.yml
```

## Was wird eingerichtet?

| Playbook | Inhalt |
|---|---|
| `01_text_ai_setup.yml` | Podman Container `ai-box`, GPU-Treiber (auto), Ollama, Open WebUI (Port 8080), systemd User-Service, Desktop-Eintrag |
| `02_image_ai_setup.yml` | ComfyUI, PyTorch (GPU-angepasst), Modell-Verzeichnisse, Start-Skript (Port 8188) |

## Dienste verwalten

```bash
# Starten
systemctl --user start ai-box

# Stoppen
systemctl --user stop ai-box

# Beim Login automatisch starten
systemctl --user enable ai-box

# Status prüfen
systemctl --user status ai-box

# Logs ansehen
podman logs ai-box
```

## Nach der Installation

| Dienst | URL |
|---|---|
| Open WebUI (Chat) | http://localhost:8080 |
| ComfyUI (Bilder) | http://localhost:8188 |
| Ollama API | http://localhost:11434 |

## Modelle laden

```bash
# Text-Modelle (direkt im Container)
podman exec ai-box ollama pull llama3.2
podman exec ai-box ollama pull mistral
podman exec ai-box ollama pull deepseek-r1:14b

# Bild-Modell herunterladen (SDXL ~6 GB)
# In 02_image_ai_setup.yml: download_example_model: true setzen
```

## Headless-Server

Auf einem Server ohne Desktop-Umgebung (kein `DISPLAY`/`WAYLAND_DISPLAY`) werden:
- ❌ Desktop-Eintrag (`.desktop`) **nicht** erstellt
- ❌ Desktop-Datenbank **nicht** aktualisiert
- ✅ systemd User-Service **wird** eingerichtet
- ✅ Container läuft dauerhaft im Hintergrund

Der Zugriff erfolgt über SSH-Tunnel oder direkt über die Server-IP:

```bash
# SSH-Tunnel vom lokalen Rechner zum Server
ssh -L 8080:localhost:8080 -L 11434:localhost:11434 user@server
# Dann im Browser: http://localhost:8080
```

## Wichtige Variablen

Beide Playbooks können in der `vars`-Sektion angepasst werden:

| Variable | Standard | Beschreibung |
|---|---|---|
| `container_name` | `ai-box` | Name des Podman-Containers |
| `ollama_port` | `11434` | Ollama API Port |
| `webui_port` | `8080` | Open WebUI Port |
| `amd_gfx_version` | `11.0.0` | AMD GPU GFX-Version (z. B. `10.3.0` für RX 6900 XT) |
| `data_dir` | `~/.local/share/ai-box` | Persistentes Datenverzeichnis |
