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
```

## Was wird eingerichtet?

| Playbook | Inhalt |
|---|---|
| `01_text_ai_setup.yml` | Distrobox Container `ai-box`, ROCm, Ollama, Open WebUI (Port 8080) |
| `02_image_ai_setup.yml` | ComfyUI, PyTorch-ROCm, Modell-Verzeichnisse, Start-Skript (Port 8188) |

## Nach der Installation

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

Beide Playbooks können oben in der `vars`-Sektion angepasst werden:

- `container_name`: Name des Distrobox-Containers (Standard: `ai-box`)
- `hsa_override`: GPU-Ziel-Architektur (RX 6900 XT = `10.3.0`)
- `ollama_keep_alive`: `0` = VRAM sofort freigeben (empfohlen bei gleichzeitigem Betrieb)
- `download_example_model`: `true` um SDXL automatisch herunterzuladen
