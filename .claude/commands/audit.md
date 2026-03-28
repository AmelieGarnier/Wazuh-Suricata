---
description: Audite les runbooks Wazuh et Suricata et liste tous les problèmes trouvés (versions, commandes, sécurité, manques)
allowed-tools: Read, Bash(grep:*), Bash(cat:*)
---

Lis les fichiers `docs/wazuh-runbook.md` et `docs/suricata-runbook.md` et effectue un audit technique complet.

Pour chaque fichier, vérifie :

## Checklist d'audit

### Versions et URLs
- [ ] Les URLs `packages.wazuh.com` correspondent bien à la version déclarée en header
- [ ] La version du module filebeat (`wazuh-filebeat-X.X`) est correcte pour la version Wazuh
- [ ] Les versions des agents (`.deb`, `.msi`) correspondent à la version déclarée
- [ ] Les URLs de téléchargement des règles Suricata sont accessibles et à jour

### Configuration
- [ ] `vm.max_map_count=262144` est documenté avant le démarrage de wazuh-indexer
- [ ] Le port `55000` (API Manager) est listé dans les prérequis réseau
- [ ] Le bloc `<indexer>` dans `ossec.conf` contient la section `<ssl>` complète
- [ ] Le fichier `wazuh.yml` (connexion Dashboard → API) est configuré
- [ ] La section `default-rule-path` est présente dans suricata.yaml
- [ ] `rule-files: - suricata.rules` est documenté

### Sécurité
- [ ] Pas de `chown root:root` sur des fichiers qui appartiennent à d'autres services
- [ ] La désactivation du dépôt Wazuh post-install est documentée
- [ ] Le changement de mot de passe `admin/admin` est mentionné
- [ ] Les permissions des certificats (500/400) sont correctes

### Performances
- [ ] La désactivation GRO/LRO (`ethtool -K ens192 gro off lro off`) est documentée
- [ ] Les offloads sont rendus persistants (systemd ou network config)

### Cohérence
- [ ] Le titre "IDS/IPS" de suricata-runbook correspond au contenu (section IPS présente ?)
- [ ] Pas de packages installés mais jamais utilisés
- [ ] Les descriptions des chemins de fichiers sont exactes

## Format de sortie

Pour chaque problème trouvé, afficher :
- **Sévérité** : 🔴 Critique | 🟠 Erreur | 🟡 Manque
- **Fichier et section**
- **Description du problème**
- **Correction suggérée**

Terminer par un résumé : nombre de problèmes par sévérité et score global.

