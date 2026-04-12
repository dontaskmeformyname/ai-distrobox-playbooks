# Local AI – Ansible Playbooks (Podman, GPU-agnostisch)

Ansible-Playbooks zur automatischen Einrichtung einer lokalen KI-Umgebung mit **Podman**-Containern.
Funktioniert sowohl auf einem **Headless-Server** als auch auf einem **Desktop-System**.

## Zielsysteme

Das Playbook läuft standardmäßig auf allen Hosts der Gruppe `ai_hosts`.
Diese Gruppe ist im Inventory als Sammelgruppe definiert und kann beliebige Untergruppen enthalten.
Über `--limit` kann die Ausführung auf einzelne Hosts oder Untergruppen eingeschränkt werden.

## Inventory-Struktur

Gruppen können über `:children` zu Mitgliedern anderer Gruppen werden:

```ini
[desktops]
ws-a ansible_host=192.168.1.10 ansible_user=alex
ws-b ansible_host=192.168.1.11 ansible_user=alex

[servers]
ai-01 ansible_host=192.168.1.20 ansible_user=alex
ai-02 ansible_host=192.168.1.21 ansible_user=alex

[ai_hosts:children]
desktops
servers

[all:vars]
ansible_python_interpreter=/usr/bin/python3
```

## Ausführung

```bash
# Alle Hosts in ai_hosts (Standard)
ansible-playbook -i inventory 01_text_ai_setup.yml --ask-become-pass

# Einzelner Host
ansible-playbook -i inventory 01_text_ai_setup.yml --limit gpu-1 --ask-become-pass

# Einzelne Gruppe
ansible-playbook -i inventory 01_text_ai_setup.yml --limit servers --ask-become-pass

# Mehrere explizite Hosts
ansible-playbook -i inventory 01_text_ai_setup.yml --limit 'gpu-1,gpu-2' --ask-become-pass
```

## Test-Ausführung

Mit `--check` wird das Playbook im Trockenlauf ausgeführt – es werden keine Änderungen vorgenommen:

```bash
# Trockenlauf für alle ai_hosts
ansible-playbook -i inventory 01_text_ai_setup.yml --check --ask-become-pass

# Trockenlauf mit ausführlicher Ausgabe
ansible-playbook -i inventory 01_text_ai_setup.yml --check --diff --ask-become-pass

# Trockenlauf auf einzelnem Host
ansible-playbook -i inventory 01_text_ai_setup.yml --check --limit gpu-1 --ask-become-pass

# Nur Syntax prüfen (kein SSH-Zugriff nötig)
ansible-playbook -i inventory 01_text_ai_setup.yml --syntax-check

# Inventory und Gruppenmitgliedschaften anzeigen
ansible -i inventory ai_hosts --list-hosts
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
ansible-playbook -i inventory 01_text_ai_setup.yml --ask-become-pass

# 2. Bild-KI: ComfyUI + PyTorch (GPU-angepasst)
ansible-playbook -i inventory 02_image_ai_setup.yml --ask-become-pass
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
