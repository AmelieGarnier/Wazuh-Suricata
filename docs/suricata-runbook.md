# Runbook Suricata — IDS/IPS

> **Environnement :** Debian 12 — IP : `192.168.111.62` — Interface : `ens192`
> **Version Suricata :** 6.0.10 — Règles : Emerging Threats Open (46 334 règles)

---

## Table des matières

1. [Prérequis](#1-prérequis)
2. [Installation](#2-installation)
3. [Configuration](#3-configuration)
4. [Gestion des règles](#4-gestion-des-règles)
5. [Intégration avec Wazuh](#5-intégration-avec-wazuh)
6. [Démarrage et automatisation](#6-démarrage-et-automatisation)
7. [Tests de détection](#7-tests-de-détection)
8. [Opérations courantes](#8-opérations-courantes)
9. [Troubleshooting](#9-troubleshooting)

---

## 1. Prérequis

### Configuration réseau

```bash
ip addr show
```

| Interface | IP | État |
|-----------|-----|------|
| `ens192` | 192.168.111.62/24 | UP |
| `lo` | 127.0.0.1 | UP |

### Mettre à jour le système

```bash
apt update && apt upgrade -y
```

---

## 2. Installation

### Installer Suricata et ses dépendances

```bash
# Suricata + jq (traitement JSON)
apt install -y suricata jq

# Gestionnaire de dépôts
apt install -y software-properties-common

# Python (outils d'intégration)
apt install -y python3-pip
```

**Versions installées :**

| Paquet | Version |
|--------|---------|
| suricata | 6.0.10 |
| jq | 1.6-2.1 |
| software-properties-common | 0.99.30-4.1 |

> Le paquet `linux-image-6.1.0-25-amd64` est installé automatiquement comme dépendance pour la capture réseau AF_PACKET.

---

## 3. Configuration

### 3.1 Fichier principal

```bash
micro /etc/suricata/suricata.yaml
```

### 3.2 Configuration AF_PACKET (capture réseau)

Localiser la section `af-packet` (lignes ~587-623) :

```yaml
af-packet:
  - interface: ens192
    threads: 2
    cluster-id: 99
    cluster-type: cluster_flow
    defrag: yes
```

> **AF_PACKET** permet la capture haute performance directement depuis le noyau Linux sans copie mémoire.

### 3.3 Définition des zones réseau

Section `vars` (lignes ~15-51) :

```yaml
vars:
  address-groups:
    HOME_NET: "[192.168.111.0/24]"
    EXTERNAL_NET: "!$HOME_NET"

    HTTP_SERVERS: "$HOME_NET"
    SMTP_SERVERS: "$HOME_NET"
    SQL_SERVERS:  "$HOME_NET"
    DNS_SERVERS:  "$HOME_NET"
    TELNET_SERVERS: "$HOME_NET"
```

### 3.4 Configuration des ports de services

```yaml
  port-groups:
    HTTP_PORTS:  "80"
    SHELLCODE_PORTS: "!80"
    ORACLE_PORTS: 1521
    SSH_PORTS:   22
    DNP3_PORTS:  20000
    MODBUS_PORTS: 502
    FILE_DATA_PORTS: "[$HTTP_PORTS,110,143]"
    FTP_PORTS:   21
    VXLAN_PORTS: 4789
    TEREDO_PORTS: 3544
```

### 3.5 Configuration des logs EVE-JSON

Section `outputs` (lignes ~85-118) :

```yaml
outputs:
  - eve-log:
      enabled: yes
      filetype: regular
      filename: /var/log/suricata/eve.json
      types:
        - alert:
            payload: yes
            payload-printable: yes
            metadata: yes
        - http:
            extended: yes
        - dns:
            query: yes
            answer: yes
        - tls:
            extended: yes
        - files:
            force-magic: no
        - smtp: {}
        - ssh: {}
        - stats:
            totals: yes
            threads: no
            deltas: no
        - flow: {}
```

---

## 4. Gestion des règles

### 4.1 Activer les règles Emerging Threats Open

```bash
suricata-update enable-source et/open
```

### 4.2 Sources de règles disponibles

```bash
suricata-update list-sources
```

| Source | Type | Description |
|--------|------|-------------|
| `et/open` | Gratuite | Emerging Threats Open — 46 334 règles |
| `et/pro` | Payante | Emerging Threats Pro — couverture étendue |
| `oisf/trafficid` | Gratuite | Identification de protocoles |

### 4.3 Mettre à jour les règles

```bash
suricata-update
```

**Résultat attendu :**
- Téléchargement : ~5,2 Mo depuis `emergingthreats.net`
- Règles chargées : 62 149
- Règles actives déployées : **46 334**

### 4.4 Valider la configuration

```bash
suricata -T -c /etc/suricata/suricata.yaml -v
```

> Doit retourner `Configuration provided was successfully loaded.`

### 4.5 Architecture des fichiers de règles

```
/var/lib/suricata/rules/
├── suricata.rules          # Règles consolidées (46 334)
└── classification.config   # Classification des alertes

/etc/suricata/suricata.yaml # Configuration principale
/var/log/suricata/
├── eve.json                # Événements structurés JSON
└── stats.log               # Statistiques de performance
```

---

## 5. Intégration avec Wazuh

### 5.1 Configurer Wazuh pour lire les logs Suricata

```bash
micro /var/ossec/etc/ossec.conf
```

Ajouter dans `<ossec_config>` :

```xml
<localfile>
  <log_format>json</log_format>
  <location>/var/log/suricata/eve.json</location>
</localfile>
```

### 5.2 Résoudre les erreurs de décodage JSON

Si des erreurs apparaissent dans `/var/ossec/logs/ossec.log` liées à des événements `stats` volumineux :

```bash
micro /var/ossec/etc/internal_options.conf
```

Modifier ou ajouter :

```
analysisd.decoder_order_size=512
```

### 5.3 Corriger les permissions et redémarrer

```bash
chmod 644 /var/log/suricata/eve.json
chown root:root /var/log/suricata/eve.json
systemctl restart wazuh-manager
```

### 5.4 Vérifier l'intégration

```bash
# Vérifier que Wazuh lit le fichier
tail -f /var/ossec/logs/ossec.log | grep suricata

# Inspecter les événements JSON Suricata
tail -50 /var/log/suricata/eve.json | jq
```

---

## 6. Démarrage et automatisation

### 6.1 Activer le démarrage automatique

```bash
systemctl enable suricata
```

### 6.2 Démarrer et vérifier le service

```bash
systemctl start suricata
systemctl status suricata
```

### 6.3 Automatiser la mise à jour des règles (cron)

```bash
crontab -e
```

Ajouter :

```cron
0 2 * * * /usr/bin/suricata-update && systemctl reload suricata
```

> Met à jour les règles chaque nuit à 2h00 et recharge Suricata.

Vérifier le cron :

```bash
crontab -l
```

### 6.4 Rotation des logs

```bash
micro /etc/logrotate.d/suricata
```

```
/var/log/suricata/*.log /var/log/suricata/*.json {
    daily
    rotate 7
    compress
    missingok
    notifempty
    postrotate
        systemctl reload suricata 2>/dev/null || true
    endscript
}
```

---

## 7. Tests de détection

### 7.1 Générer des événements de test

```bash
# Test de connectivité réseau (génère des événements DNS/HTTP)
curl -I http://testmynids.org/uid/index.html

# Test avec une règle de détection connue (GPL ATTACK_RESPONSE)
curl http://www.testmyids.com
```

### 7.2 Vérifier les alertes dans eve.json

```bash
# Alertes en temps réel
tail -f /var/log/suricata/eve.json | jq 'select(.event_type == "alert")'

# Derniers événements
tail -50 /var/log/suricata/eve.json | jq '{type: .event_type, src: .src_ip, dest: .dest_ip, alert: .alert.signature}'
```

### 7.3 Vérifier dans Wazuh Dashboard

1. Aller dans **Threat Hunting** → **Events**
2. Filtrer par `data.event_type: alert`
3. Les alertes Suricata apparaissent avec les métadonnées complètes (signature, catégorie, sévérité)

---

## 8. Opérations courantes

### État du service

```bash
systemctl status suricata
```

### Logs en temps réel

```bash
# Toutes les alertes
tail -f /var/log/suricata/eve.json | jq 'select(.event_type == "alert")'

# Statistiques
tail -f /var/log/suricata/stats.log
```

### Inspecter les règles déployées

```bash
ls -la /var/lib/suricata/rules/
wc -l /var/lib/suricata/rules/suricata.rules
```

### Rechercher une règle spécifique

```bash
grep -i "ssh" /var/lib/suricata/rules/suricata.rules | head -20
grep -i "dns" /var/lib/suricata/rules/suricata.rules | head -20
```

### Mettre à jour les règles manuellement

```bash
suricata-update
systemctl reload suricata
```

### Recharger la configuration sans interruption

```bash
systemctl reload suricata
```

### Redémarrage complet

```bash
systemctl restart suricata
```

---

## 9. Troubleshooting

### Suricata ne démarre pas

```bash
# Vérifier les logs système
journalctl -u suricata -n 50

# Valider la configuration
suricata -T -c /etc/suricata/suricata.yaml -v

# Vérifier les permissions
ls -la /var/log/suricata/
```

---

### Aucune alerte dans eve.json

```bash
# Vérifier que Suricata écoute sur la bonne interface
ps aux | grep suricata

# Vérifier que les règles sont chargées
grep "rules loaded" /var/log/suricata/suricata.log

# S'assurer que l'interface est UP
ip link show ens192

# Tester manuellement
curl http://www.testmyids.com
tail -f /var/log/suricata/eve.json | jq
```

---

### Wazuh ne reçoit pas les alertes Suricata

```bash
# 1. Vérifier la configuration localfile dans ossec.conf
grep -A 4 "eve.json" /var/ossec/etc/ossec.conf

# 2. Vérifier les permissions du fichier eve.json
ls -la /var/log/suricata/eve.json
# → doit être lisible par root (644 minimum)

# 3. Vérifier les erreurs dans les logs Wazuh
tail -f /var/ossec/logs/ossec.log | grep -i "suricata\|eve\|json"

# 4. Corriger les permissions si nécessaire
chmod 644 /var/log/suricata/eve.json
chown root:root /var/log/suricata/eve.json
systemctl restart wazuh-manager
```

---

### Erreurs de décodage JSON dans Wazuh

```
ERROR: JSON decoder size exceeded
```

```bash
micro /var/ossec/etc/internal_options.conf
# Modifier : analysisd.decoder_order_size=512

systemctl restart wazuh-manager
```

---

### Performances dégradées (CPU élevé)

```bash
# Vérifier la charge CPU de Suricata
top -p $(pgrep suricata)

# Vérifier les statistiques
cat /var/log/suricata/stats.log | tail -50

# Augmenter les threads AF_PACKET dans suricata.yaml si nécessaire
micro /etc/suricata/suricata.yaml
# → af-packet: threads: 4  (selon le nombre de cœurs disponibles)

systemctl restart suricata
```

---

### Faux positifs — Désactiver une règle

```bash
# Trouver le SID de la règle
tail -f /var/log/suricata/eve.json | jq 'select(.event_type == "alert") | .alert'

# Désactiver le SID (ex: SID 2100498)
echo "disable-rule 2100498" >> /etc/suricata/disable.conf
suricata-update
systemctl reload suricata
```

---

## Références

| Ressource | URL |
|-----------|-----|
| Documentation Suricata | https://suricata.io/docs/ |
| Wiki Emerging Threats | https://doc.emergingthreats.net/ |
| Documentation Wazuh | https://documentation.wazuh.com/ |
| Forum Suricata | https://forum.suricata.io/ |
