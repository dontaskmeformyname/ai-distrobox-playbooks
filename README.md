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

## HuggingFace Token (optional, empfohlen)

Ein HuggingFace-Token erhöht die Rate-Limits beim Modell-Download und ist für einige Modelle
pflicht (z. B. Llama 3 bei erstmaliger Nutzung). Das Token wird **lokal** gespeichert und
**nie nach GitHub gepusht**.

### Einmalig einrichten

```bash
cp secrets.yml.example secrets.yml
nano secrets.yml   # hf_token: "hf_DEINTOKEN"
```

Token erstellen: <https://huggingface.co/settings/tokens>

### Sicherheit

| Datei | Im Repo? | Zweck |
|---|---|---|
| `secrets.yml.example` | ✅ ja | Vorlage, kein echter Token |
| `secrets.yml` | ❌ nein (`.gitignore`) | Dein echter Token |

Das Playbook gibt beim Start eine **Warnung** aus, wenn `secrets.yml` fehlt
oder `hf_token` leer ist – es läuft aber trotzdem durch.
Den Token nachträglich setzen und Playbook neu ausführen genügt
– der laufende Container wird dabei nicht neu erstellt.

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

## GPU manuell identifizieren

Die GPU-Erkennung erfolgt automatisch via `lspci` auf dem **Host** (nicht im Container).
Bei mehreren GPUs im System wird zu Beginn eine Liste aller gefundenen GPUs ausgegeben.

### Alle GPUs im System anzeigen

```bash
# Alle Grafikkarten anzeigen (VGA, 3D, Display Controller)
lspci | grep -E 'VGA|3D|Display'

# Nach Hersteller filtern
lspci | grep -i nvidia          # NVIDIA
lspci | grep -i amd             # AMD / Radeon
lspci | grep -iE 'intel.*(graphics|vga|uhd|iris|arc)'  # Intel

# DRI Render-Nodes anzeigen (AMD/Intel, Index 0, 1, 2 ...)
ls -1 /dev/dri/renderD*
```

Beispielausgabe mit zwei GPUs (iGPU + dGPU):

```
00:02.0 VGA compatible controller: Intel Corporation UHD Graphics 630
01:00.0 VGA compatible controller: Advanced Micro Devices [AMD] Navi 21 [Radeon RX 6900 XT]
```

### Manuelle GPU-Auswahl

Bei mehreren GPUs im System oder falls die automatische Erkennung fehlschlägt,
kann die gewünschte GPU über `gpu_override` festgelegt werden.

Das Format ist `hersteller` oder `hersteller:index` (Index beginnt bei 0):

| Wert | Bedeutung |
|---|---|
| `""` | Automatisch (Standard) |
| `amd` | Erste AMD GPU (Index 0) |
| `amd:0` | Erste AMD GPU (explizit) |
| `amd:1` | Zweite AMD GPU |
| `nvidia` | Erste NVIDIA GPU (Index 0) |
| `nvidia:1` | Zweite NVIDIA GPU |
| `intel` | Intel GPU |
| `cpu` | CPU-only, keine GPU |

**Option 1 – direkt in `01_text_ai_setup.yml` unter `vars`:**

```yaml
vars:
  gpu_override: "amd:1"   # zweite AMD GPU verwenden
```

**Option 2 – per CLI ohne Datei zu ändern:**

```bash
# Erste AMD GPU (Standard bei einem AMD-System)
ansible-playbook -i inventory 01_text_ai_setup.yml --ask-become-pass -e gpu_override=amd

# Zweite AMD GPU (z.B. RX 6900 XT wenn iGPU auf Index 0 liegt)
ansible-playbook -i inventory 01_text_ai_setup.yml --ask-become-pass -e gpu_override=amd:1

# Erste NVIDIA GPU
ansible-playbook -i inventory 01_text_ai_setup.yml --ask-become-pass -e gpu_override=nvidia

# Zweite NVIDIA GPU
ansible-playbook -i inventory 01_text_ai_setup.yml --ask-become-pass -e gpu_override=nvidia:1

# CPU-only erzwingen
ansible-playbook -i inventory 01_text_ai_setup.yml --ask-become-pass -e gpu_override=cpu
```

### Mehrere GPUs – Priorität bei automatischer Erkennung

Wenn mehrere GPU-Hersteller erkannt werden (z.B. Intel iGPU + AMD dGPU),
gelten folgende Prioritäten:

```
NVIDIA > AMD > Intel > CPU-only
```

Das Playbook gibt in diesem Fall eine Warnung aus und empfiehlt `gpu_override`.

## Unterstützte GPUs

| GPU-Hersteller | Beschleunigung | Erkennungsmethode |
|---|---|---|
| NVIDIA | CUDA | `lspci` (grep nvidia) |
| AMD | ROCm | `lspci` (grep amd/radeon) |
| Intel | oneAPI / Level Zero | `lspci` (grep intel graphics) |
| Kein GPU | CPU-only Fallback | automatisch |

Die GPU-Erkennung erfolgt **vollautomatisch** zu Beginn des Playbooks auf dem Host.
Die benötigten Treiber/Bibliotheken werden entsprechend im Container installiert.

## System-Voraussetzungen

- Linux-Host (x86_64)
- Podman installiert (`podman --version`)
- Ansible installiert (`ansible --version`)
- `pciutils` installiert (`lspci` muss verfügbar sein)
- Für AMD: `/dev/kfd` und `/dev/dri/renderD*` vorhanden (AMDGPU-Treiber geladen)
- Für NVIDIA: NVIDIA Container Toolkit installiert
- Für Intel: `/dev/dri/renderD128` vorhanden

## Installation

```bash
# Optional: HuggingFace Token hinterlegen (empfohlen)
cp secrets.yml.example secrets.yml
nano secrets.yml   # hf_token: "hf_DEINTOKEN"

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
| `amd_gfx_version` | `""` | AMD GPU GFX-Version für ROCm (leer = auto) |
| `data_dir` | `~/.local/share/ai-box` | Persistentes Datenverzeichnis |
| `gpu_override` | `""` (auto) | GPU auswählen: `amd`, `amd:1`, `nvidia`, `nvidia:1`, `intel`, `cpu` |
| `hf_token` | `""` | HuggingFace Token – via `secrets.yml` setzen, nie per CLI übergeben |
