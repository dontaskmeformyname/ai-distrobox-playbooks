# Local AI – Ansible Playbooks (Podman, GPU-agnostisch)

Ansible-Playbooks zur automatischen Einrichtung einer lokalen KI-Umgebung mit **Podman**-Containern.
Funktioniert sowohl auf einem **Headless-Server** als auch auf einem **Desktop-System**.

## Zielsysteme

Das Playbook ist **gruppenagnostisch**.
Es läuft auf beliebigen Hosts oder Gruppen aus dem Inventory und erwartet keine festen Gruppennamen im Playbook.
Ansible stellt dafür die Gruppe `all` als Sammelgruppe für alle Inventory-Hosts bereit.

## Sichere Ausführung

Die Zielmenge wird über `--limit` eingeschränkt.
Zusätzlich läuft das Playbook seriell mit `serial: 1`, damit Hosts nacheinander verarbeitet werden.
Wenn mehr als ein Host betroffen ist, bricht das Playbook standardmäßig ab.
Mehrere Hosts sind nur mit ausdrücklicher Freigabe über `-e allow_multi_host=true` erlaubt.

### Beispiele

```bash
# Einzelner Host
ansible-playbook -i inventory 01_text_ai_setup.yml --limit gpu-1 --ask-become-pass

# Einzelne Gruppe
ansible-playbook -i inventory 01_text_ai_setup.yml --limit servers --ask-become-pass -e allow_multi_host=true

# Mehrere explizite Hosts
ansible-playbook -i inventory 01_text_ai_setup.yml --limit 'gpu-1,gpu-2' --ask-become-pass -e allow_multi_host=true
```

## Beispiel-Inventory

```ini
[desktops]
ws-a ansible_host=192.168.1.10 ansible_user=alex
ws-b ansible_host=192.168.1.11 ansible_user=alex

[servers]
ai-01 ansible_host=192.168.1.20 ansible_user=alex
ai-02 ansible_host=192.168.1.21 ansible_user=alex

[all:vars]
ansible_python_interpreter=/usr/bin/python3
```

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
ansible-playbook -i inventory 01_text_ai_setup.yml --limit <host|gruppe> --ask-become-pass

# 2. Bild-KI: ComfyUI + PyTorch (GPU-angepasst)
ansible-playbook -i inventory 02_image_ai_setup.yml --limit <host|gruppe> --ask-become-pass
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
podman exec ai-box ollama pull llama3.2
podman exec ai-box ollama pull mistral
podman exec ai-box ollama pull deepseek-r1:14b
```

## Headless-Server

Auf einem Server ohne Desktop-Umgebung (kein `DISPLAY` oder `WAYLAND_DISPLAY`) werden kein Desktop-Eintrag und keine Desktop-Datenbank erstellt.
Der systemd User-Service wird trotzdem eingerichtet und der Container kann dauerhaft im Hintergrund laufen.

```bash
ssh -L 8080:localhost:8080 -L 11434:localhost:11434 user@server
# Danach im Browser: http://localhost:8080
```

## Wichtige Variablen

| Variable | Standard | Beschreibung |
|---|---|---|
| `container_name` | `ai-box` | Name des Podman-Containers |
| `ollama_port` | `11434` | Ollama API Port |
| `webui_port` | `8080` | Open WebUI Port |
| `amd_gfx_version` | `11.0.0` | AMD GPU GFX-Version |
| `data_dir` | `~/.local/share/ai-box` | Persistentes Datenverzeichnis |
| `allow_multi_host` | `false` | Erlaubt bewusste Mehrfachausführung |
