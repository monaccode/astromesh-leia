# astromesh-leia

A Claude Code plugin that provides a natural-language interface for creating, deploying, and managing AI agents on astromesh-nexus clusters. Named after Leia, a lemon beagle, this plugin takes you from business idea to deployed WhatsApp agent in minutes -- no Kubernetes expertise required.

## Architecture

```mermaid
graph LR
    subgraph "Claude Code CLI"
        subgraph "astromesh-leia Plugin"
            CMD[Commands]
            AGT[Agents]
            TPL[Templates]
            SCH[Schemas]
        end
    end

    subgraph "Kubernetes Cluster"
        API[Nexus API]
        TNS[Tenant Namespaces]
        NOD[astromesh-nodes]
        API --> TNS
        TNS --> NOD
    end

    subgraph "External Services"
        WA[Meta WhatsApp]
        OLL[Ollama]
        LLM[Cloud LLMs]
    end

    CMD --> API
    AGT --> API
    TPL --> CMD
    SCH --> CMD
    NOD --> WA
    NOD --> OLL
    NOD --> LLM
```

## Quick Start

```bash
git clone https://github.com/monaccode/astromesh-leia.git
claude plugins add ./astromesh-leia
```

## Command Reference

| Command | Description |
|---------|-------------|
| `/leia` | Conversational entry point |
| `/leia create` | Create agent (wizard or NL) |
| `/leia deploy` | Deploy agent YAML to nexus |
| `/leia status` | CLI dashboard |
| `/leia logs` | Agent log viewer |
| `/leia test` | Interactive agent testing |
| `/leia templates` | Browse templates |
| `/leia config` | Connection management |
| `/leia bootstrap` | Cluster lifecycle |
| `/leia teardown` | Destroy cluster |

## Templates

| Template | Description |
|----------|-------------|
| `customer-support` | Customer support agent with FAQ handling and escalation |
| `restaurant-booking` | Restaurant reservation and menu inquiry agent |
| `ecommerce-assistant` | Product search, cart management, and order tracking agent |
| `appointment-scheduler` | Calendar-aware appointment booking agent |
| `lead-qualifier` | Lead scoring and qualification conversational agent |
| `onboarding-guide` | User onboarding and product walkthrough agent |

## Subagents

| Agent | Model | Role |
|-------|-------|------|
| `interpreter` | Sonnet | Parses natural-language intent into structured specs |
| `architect` | Opus | Designs agent architecture and generates manifests |
| `operator` | Sonnet | Executes kubectl and Nexus API operations |
| `tester` | Sonnet | Runs conversation simulations and validates behavior |
| `doctor` | Sonnet | Diagnoses failures and suggests remediation |

## Documentation

| Document | Path |
|----------|------|
| Tutorials | [docs/tutorials/](docs/tutorials/) |
| Plugin Development | [docs/](docs/) |

## Version Compatibility

| Leia | Nexus | Astromesh |
|------|-------|-----------|
| 0.1.x | 0.1.x | 0.18+ |

## License

Licensed under the Apache License, Version 2.0. See [LICENSE](LICENSE) for details.
