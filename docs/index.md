# p4n4

**p4n4** is a self-hosted, open-source platform for IoT, GenAI, and Edge AI — assembled from
best-of-breed open-source services and wired together with Docker Compose.

## What's in the box

| Stack | Services |
|-------|---------|
| [IoT](iot-stack.md) | Eclipse Mosquitto · Node-RED · InfluxDB · Grafana |
| [AI](ai-stack.md) | Ollama · Letta · n8n |
| [Edge](edge-stack.md) | Edge Impulse Linux Runner |

## Quick start

```bash
pip install p4n4
p4n4 init my-project
cd my-project
p4n4 up
```

## Repository map

```
p4n4/                       (monorepo — you are here)
├── docker/
│   ├── iot/                ← IoT stack (p4n4-iot)
│   ├── ai/                 ← GenAI stack (p4n4-ai)
│   └── edge/               ← Edge AI stack (p4n4-edge)
├── shared/
│   ├── lib/                ← Shared library (p4n4-lib)
│   ├── hw/                 ← Hardware designs + RPi5 scripts (p4n4-hw)
│   └── templates/          ← Community templates (p4n4-templates)
├── client/
│   ├── cli/                ← Python CLI — pip install p4n4 (p4n4-cli)
│   ├── api/                ← REST API gateway (p4n4-api)
│   └── dashboard/          ← Web dashboard (p4n4-dashboard)
├── demo/
│   └── emu/                ← Hardware emulator for workstation dev (p4n4-emu)
└── docs/                   ← this documentation (p4n4-docs)
```
