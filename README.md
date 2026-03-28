# Wazuh + Suricata — Documentation Technique

[![Wazuh](https://img.shields.io/badge/Wazuh-4.13-0070C0?style=flat-square&logo=wazuh&logoColor=white)](https://documentation.wazuh.com/current/index.html)
[![Suricata](https://img.shields.io/badge/Suricata-6.0.10-EF6C00?style=flat-square&logo=suricata&logoColor=white)](https://suricata.io/)
[![Debian](https://img.shields.io/badge/Debian-13-A81D33?style=flat-square&logo=debian&logoColor=white)](https://www.debian.org/)
[![OpenSearch](https://img.shields.io/badge/OpenSearch-Indexer-005EB8?style=flat-square&logo=opensearch&logoColor=white)](https://opensearch.org/)
[![VirusTotal](https://img.shields.io/badge/VirusTotal-Intégré-394EFF?style=flat-square&logo=virustotal&logoColor=white)](https://www.virustotal.com/)
[![SIEM](https://img.shields.io/badge/SIEM-Single--Node-2E7D32?style=flat-square&logo=shield&logoColor=white)](Runbooks/wazuh-runbook.md)
[![FIM](https://img.shields.io/badge/FIM-Whodata-6A1B9A?style=flat-square&logo=files&logoColor=white)](Runbooks/wazuh-runbook.md#5-configuration-fim)
[![Active Response](https://img.shields.io/badge/Active_Response-Brute_Force-C62828?style=flat-square&logo=security&logoColor=white)](Runbooks/wazuh-runbook.md#6-active-response--brute-force)
[![Docs](https://img.shields.io/badge/Docs-Runbooks-37474F?style=flat-square&logo=readthedocs&logoColor=white)](Runbooks/)

> Déploiement d'un SIEM **Wazuh 4.13** intégré avec **Suricata IDS** sur Debian 13.
> Architecture single-node avec agents Linux et Windows.

---

## Contenu du dépôt

```
Wazuh-Suricata/
├── Documentations/
│   ├── AMELIE_GARNIER_WAZUH.pdf        # Rapport technique Wazuh
│   └── AMELIE_GARNIER_SURICATA.pdf     # Rapport technique Suricata
└── Runbooks/
    ├── wazuh-runbook.md                # Guide d'installation et d'exploitation Wazuh
    └── suricata-runbook.md             # Guide d'installation et d'exploitation Suricata
```

---

## Runbooks

| Runbook | Description |
|---------|-------------|
| [Wazuh](Runbooks/wazuh-runbook.md) | Installation, configuration, agents, FIM, Active Response, troubleshooting |
| [Suricata](Runbooks/suricata-runbook.md) | Installation, configuration AF_PACKET, règles ET Open, intégration Wazuh, troubleshooting |

## Documentations PDF

| Document | Description |
|----------|-------------|
| [Rapport Wazuh](Documentations/AMELIE_GARNIER_WAZUH.pdf) | Documentation technique complète — SIEM Wazuh |
| [Rapport Suricata](Documentations/AMELIE_GARNIER_SURICATA.pdf) | Documentation technique complète — IDS Suricata |

---

## Architecture

```
┌──────────────────────────────────────────────┐
│            Serveur Debian 13                 │
│                                              │
│  ┌──────────────────┐  ┌──────────────────┐  │
│  │   Wazuh Stack    │  │  Suricata IDS    │  │
│  │                  │  │                  │  │
│  │  ▸ Manager       │◄─│  Interface ens192│  │
│  │  ▸ Indexer       │  │  46 334 règles   │  │
│  │  ▸ Dashboard     │  │  → eve.json      │  │
│  │  ▸ Filebeat      │  └──────────────────┘  │
│  └──────────────────┘                        │
└──────────────────────────────────────────────┘
              ▲                ▲
              │                │
       ┌──────┴─────┐   ┌──────┴─────┐
       │ Agent      │   │ Agent      │
       │ Debian 13  │   │ Windows 10 │
       └────────────┘   └────────────┘
```

---

## Stack technique

| Composant | Version | Rôle |
|-----------|---------|------|
| Wazuh Manager | 4.13.x | Analyse, corrélation, alertes |
| Wazuh Indexer | 4.13.x | Stockage et indexation (OpenSearch) |
| Wazuh Dashboard | 4.13.x | Interface web de visualisation |
| Filebeat | 8.x | Collecte et transfert des logs |
| Suricata | 6.0.10 | Détection d'intrusion réseau (IDS) |
| OS | Debian 13 | Système d'exploitation |

---

## Auteur

**Amélie Garnier**
