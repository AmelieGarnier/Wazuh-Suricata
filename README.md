# Wazuh + Suricata — Documentation Technique

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
