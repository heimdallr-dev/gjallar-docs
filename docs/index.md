# Gjallar

> **Security without chains.** See clearly, understand why, and stay free to choose. Gjallar sounds the horn when danger is real; what you do next is always yours.

Gjallar watches every image running in your Kubernetes clusters and ranks
vulnerabilities by **real risk**: whether a CVE is exploited in the wild
(CISA KEV, EPSS), whether the package actually loads at runtime, whether the
workload is exposed, and whether a fix exists. Every score says why.

| Start here | |
|---|---|
| [Philosophy](philosophy.md) | The nine freedoms every change is held to |
| [Install: WSL → Kubernetes](install-wsl.md) | From a fresh machine to a running cluster |
| [Architecture](architecture.md) | Hexagonal core, events, risk formula, security model |
| [Configuration](configuration.md) | Every setting |
| [Operations](operations.md) | Day 2, troubleshooting, network egress |
| [Interfaces](interfaces.md) | REST, MCP, CLI, Flight SQL |
| [Analytics](analytics.md) | Lakehouse tables, reports, notebooks |
| [Development](development.md) | Code layout, adding an engine, tests |
| [Status and roadmap](roadmap.md) | What is done, verified, open |

Source: [gjallar](https://github.com/heimdallr-dev/gjallar) ·
chart: [gjallar-helm](https://github.com/heimdallr-dev/gjallar-helm) ·
dashboards: [gjallar-grafana](https://github.com/heimdallr-dev/gjallar-grafana) ·
DB builder: [gjallar-vunnel](https://github.com/heimdallr-dev/gjallar-vunnel)
