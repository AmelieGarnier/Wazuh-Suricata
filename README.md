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

## Pourquoi ce projet ?

Les environnements informatiques modernes sont exposés à des menaces persistantes : brute-force SSH, exfiltration de données, malwares, intrusions réseau. Une organisation sans visibilité sur ces événements ne peut ni les détecter ni y répondre à temps.

Ce projet répond à un besoin concret :

- **Centraliser** les logs de sécurité issus de plusieurs systèmes (Linux, Windows) en un seul point d'analyse
- **Détecter** les intrusions réseau en temps réel grâce à un IDS (Suricata) couplé à 46 334 règles de détection
- **Alerter automatiquement** sur les comportements suspects : tentatives de connexion par force brute, modifications de fichiers sensibles, hash détectés par VirusTotal
- **Répondre activement** aux menaces via l'Active Response Wazuh : blocage IP automatique, suppression de fichiers malveillants

---

## Comment fonctionne la solution ?

### Architecture

```
┌──────────────────────────────────────────────┐
│            Serveur Debian 13                 │
│                                              │
│  ┌──────────────────┐  ┌──────────────────┐  │
│  │   Wazuh Stack    │  │  Suricata IDS    │  │
│  │                  │  │                  │  │
│  │  ▸ Manager       │◄─│ Interface enp0s3 │  │
│  │  ▸ Indexer       │  │  46 334 règles   │  │
│  │  ▸ Dashboard     │  │   → eve.json     │  │
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

### Flux de données

1. **Suricata** capture le trafic réseau en mode AF_PACKET sur l'interface `enp0s3` et écrit les alertes dans `/var/log/suricata/eve.json`
2. **Wazuh Manager** collecte ce fichier via un bloc `<localfile>`, décode le JSON et corrèle les événements avec ses règles
3. **Les agents Wazuh** (Linux + Windows) remontent leurs logs système, événements FIM et modifications de fichiers vers le Manager
4. **Wazuh Indexer** (OpenSearch) stocke et indexe tous les événements pour la recherche et la visualisation
5. **Le Dashboard** présente les alertes, les mappings MITRE ATT&CK, les statistiques et permet le Threat Hunting
6. **L'Active Response** déclenche automatiquement des contre-mesures : blocage d'IP avec `iptables`, suppression de fichier malveillant

### Chaîne de détection complète

```
Trafic réseau → Suricata (IDS) → eve.json → Wazuh Manager → Indexer → Dashboard
Logs système  → Agent Wazuh   ──────────────────────────────────────────────────▲
Fichiers      → FIM + Whodata ──────────────────────────────────────────────────┤
Hashes        → VirusTotal API ─────────────────────────────────────────────────┘
                                                              ↓
                                                     Active Response
                                                  (blocage IP / suppression)
```

---

## Intérêt et apports

### Capacités de détection couvertes

| Menace | Mécanisme | Règle / Module |
|--------|-----------|----------------|
| Brute-force SSH | Corrélation de tentatives | Règle 5763, MITRE T1110 |
| Intrusion réseau | Analyse de signatures IDS | Suricata ET Open (46 334 règles) |
| Modification de fichiers sensibles | FIM Whodata (auditd) | Qui / quand / quel processus |
| Malware / fichier malveillant | Hash MD5/SHA256 soumis à VT | Intégration VirusTotal |
| Exfiltration DNS | Analyse protocole DNS | Suricata rules |
| Scan de ports | Détection de paquets suspects | Suricata ET Scan rules |

### Ce que ce projet démontre

- **Déploiement complet d'un SIEM** open-source : installation, configuration, hardening
- **Intégration multi-composants** : Wazuh + Suricata + VirusTotal + OpenSearch
- **Monitoring hétérogène** : agents sur Linux et Windows dans la même console
- **Réponse automatisée aux incidents** : Active Response configurée et testée
- **Qualité de la documentation** : runbooks reproductibles, captures d'écran réelles, troubleshooting détaillé

---

## Contenu du dépôt

```
Wazuh-Suricata/
├── Documentations/
│   ├── AMELIE_GARNIER_WAZUH.pdf        # Rapport technique Wazuh
│   └── AMELIE_GARNIER_SURICATA.pdf     # Rapport technique Suricata
├── Runbooks/
│   ├── wazuh-runbook.md                # Guide d'installation et d'exploitation Wazuh
│   └── suricata-runbook.md             # Guide d'installation et d'exploitation Suricata
└── Screenshots/
    └── *.png                           # 22 captures d'écran extraites des rapports
```

---

## Runbooks

| Runbook | Description |
|---------|-------------|
| [Wazuh](Runbooks/wazuh-runbook.md) | Installation, configuration, agents, FIM, Active Response, VirusTotal, troubleshooting |
| [Suricata](Runbooks/suricata-runbook.md) | Installation, configuration AF_PACKET, règles ET Open, intégration Wazuh, troubleshooting |

## Documentations PDF

| Document | Description |
|----------|-------------|
| [Rapport Wazuh](Documentations/AMELIE_GARNIER_WAZUH.pdf) | Documentation technique complète — SIEM Wazuh |
| [Rapport Suricata](Documentations/AMELIE_GARNIER_SURICATA.pdf) | Documentation technique complète — IDS Suricata |

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
