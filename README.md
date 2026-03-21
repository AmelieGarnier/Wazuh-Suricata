# Wazuh + Suricata — Documentation Technique

Déploiement d'un SIEM Wazuh intégré avec Suricata IDS sur Debian 12.

## Documentation

| Document | Description |
|----------|-------------|
| [Runbook Wazuh](docs/wazuh-runbook.md) | Installation, configuration, agents, FIM, Active Response |
| [Runbook Suricata](docs/suricata-runbook.md) | Installation, configuration, intégration Wazuh, règles de détection |

## Architecture

```
┌─────────────────────────────────────────┐
│          Serveur Wazuh (Debian 12)       │
│          192.168.111.62/24               │
│                                          │
│  ┌─────────────┐    ┌─────────────────┐  │
│  │ Wazuh       │    │ Suricata IDS    │  │
│  │ Manager     │◄───│ (ens192)        │  │
│  │ Indexer     │    │ 46 334 règles   │  │
│  │ Dashboard   │    │ eve.json        │  │
│  └─────────────┘    └─────────────────┘  │
└─────────────────────────────────────────┘
         ▲                  ▲
         │                  │
   ┌─────┴────┐       ┌─────┴────┐
   │  Agent   │       │  Agent   │
   │  Debian  │       │  Win10   │
   └──────────┘       └──────────┘
```

## Environnement

| Composant | Version | Adresse |
|-----------|---------|---------|
| Wazuh Manager | 4.13.x | 192.168.111.62 |
| Wazuh Indexer | 4.13.x | 192.168.111.62:9200 |
| Wazuh Dashboard | 4.13.x | https://192.168.111.62 |
| Suricata | 6.0.10 | ens192 |
| OS | Debian 12 | — |

## Auteur

Amélie Garnier
