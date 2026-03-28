# Wazuh-Suricata Runbook Project

## Contexte
Runbooks de déploiement SIEM/IDS pour environnement Debian 12.
Stack : Wazuh 4.13.x (single-node) + Suricata 6.0.10 (IDS → IPS) + Shuffle SOAR.

## Structure du repo
```
docs/
├── wazuh-runbook.md       # Installation et config Wazuh all-in-one
└── suricata-runbook.md    # Installation et config Suricata IDS/IPS
.claude/
└── commands/
    ├── fix-runbooks.md    # Applique toutes les corrections d'audit
    └── audit.md           # Audite les runbooks et liste les problèmes
```

## Règles de contribution
- Toutes les commandes sont exécutées en `root` sur Debian 12
- IP du serveur : `192.168.111.62`, interface : `ens192`
- Version Wazuh cible : 4.13.x (URLs packages.wazuh.com/4.13)
- Module Filebeat : wazuh-filebeat-0.5.tar.gz
- Langue : français
- Format : Markdown avec blocs de code bash/yaml/xml

## Commandes utiles
- `/fix-runbooks` — applique les corrections et commit
- `/audit`        — liste les problèmes dans les runbooks

