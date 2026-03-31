# Runbook Suricata — IDS/IPS

> **Environnement :** Debian 13 — IP : `192.168.1.50` — Interface : `enp0s3`
> **Version Suricata :** 7.0.10 — Règles : Emerging Threats Open (46 334 règles)

---

## Présentation de Suricata

Suricata est un moteur open-source d'analyse réseau et de détection de menaces. En mode **IDS** (Intrusion Detection System), il capture et inspecte le trafic réseau en temps réel et génère des alertes au format JSON (EVE). Intégré à Wazuh, il ajoute la dimension réseau à la supervision des endpoints.

**Événements générés dans `eve.json` :**

| Type | Description |
|------|-------------|
| `alert` | Signature réseau déclenchée — à traiter en priorité |
| `dns` | Requêtes et réponses DNS avec détails (rrname, rrtype, answers) |
| `http` | Requêtes HTTP avec méthode, URI, user-agent |
| `tls` | Flux TLS/SSL avec fingerprint |
| `flow` | Résumé des flux réseau (src/dst, proto, bytes) |
| `stats` | Statistiques internes de Suricata — volumineux, peut causer des erreurs dans Wazuh |

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

### Identifier l'interface réseau

```bash
ip addr show
```

Exemple de sortie attendue :
```
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP>
    inet 192.168.1.50/24 brd 192.168.1.255 scope global enp0s3
```

| Interface | IP | État |
|-----------|-----|------|
| `enp0s3` | 192.168.1.50/24 | UP |
| `lo` | 127.0.0.1 | UP |

> ⚠️ Le nom de l'interface varie selon la machine (`enp0s3`, `ens192`, `eth0`…). Vérifier avec `ip addr show` et adapter **toutes** les commandes suivantes en conséquence.

### Mettre à jour le système

```bash
apt update && apt upgrade -y
```

---

## 2. Installation

### Installer Suricata et ses dépendances

```bash
apt install -y suricata jq
```

**Versions installées :**

| Paquet | Version |
|--------|---------|
| suricata | 7.0.10 |
| jq | 1.6+ |

### Désactiver les offloads réseau (performance AF_PACKET)

La désactivation de GRO/LRO évite les problèmes de checksum et garantit une capture complète des paquets :

```bash
# Installer ethtool si absent
apt install -y ethtool

ethtool -K enp0s3 gro off lro off
```

Résultat attendu (pas d'erreur) :
```
Actual changes:
rx-gro: off
```

Rendre persistant via systemd :

```bash
cat > /etc/systemd/system/disable-offload.service << 'EOF'
[Unit]
Description=Disable GRO/LRO on enp0s3
After=network.target

[Service]
Type=oneshot
ExecStart=/sbin/ethtool -K enp0s3 gro off lro off
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF
systemctl enable disable-offload
systemctl start disable-offload
```

> ⚠️ Remplacer `enp0s3` par le nom réel de l'interface si différent.

---

## 3. Configuration

### 3.1 Fichier principal

```bash
micro /etc/suricata/suricata.yaml
```

### 3.2 Configuration AF_PACKET (capture réseau)

Localiser la section `af-packet` :

```yaml
af-packet:
  - interface: enp0s3
    threads: 2
    cluster-id: 99
    cluster-type: cluster_flow
    defrag: yes
    use-mmap: yes
```

| Paramètre | Valeur | Description |
|-----------|--------|-------------|
| `interface` | enp0s3 | Interface réseau principale |
| `threads` | 2 | Traitement parallèle — adapter au nombre de cœurs CPU |
| `cluster-id` | 99 | Identifiant unique pour l'équilibrage de charge |
| `cluster-type` | cluster_flow | Tous les paquets d'un flux → même thread |
| `defrag` | yes | Reconstitution des paquets IP fragmentés |
| `use-mmap` | yes | Capture haute performance, réduction latence |

> **AF_PACKET** permet la capture directement depuis le noyau Linux sans copie mémoire.

### 3.3 Définition des zones réseau

Section `vars` :

```yaml
vars:
  address-groups:
    HOME_NET: "[192.168.1.0/24]"
    EXTERNAL_NET: "!$HOME_NET"

    HTTP_SERVERS: "$HOME_NET"
    SMTP_SERVERS: "$HOME_NET"
    SQL_SERVERS:  "$HOME_NET"
    DNS_SERVERS:  "$HOME_NET"
    TELNET_SERVERS: "$HOME_NET"
```

> `HOME_NET` = ton sous-réseau interne. Adapter selon l'IP du serveur (`ip a` → prendre le `/24`).

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
    GENEVE_PORTS: 6081
    VXLAN_PORTS: 4789
    TEREDO_PORTS: 3544
```

### 3.5 Chemin des règles

Vérifier que cette section est présente dans `suricata.yaml` :

```yaml
default-rule-path: /var/lib/suricata/rules
rule-files:
  - suricata.rules
```

> Sans cette configuration, `suricata-update` dépose les règles mais Suricata démarre avec **0 règle chargée**.

### 3.6 Configuration des logs EVE-JSON

Section `outputs` — vérifier que `eve-log` est activé :

```yaml
outputs:
  - eve-log:
      enabled: yes
      filetype: regular
      filename: eve.json
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

> Les logs EVE sont écrits dans `/var/log/suricata/eve.json`.

---

## 4. Gestion des règles

### 4.1 Vérifier la résolution DNS avant téléchargement

```bash
ping -c 2 google.com
```

Si la résolution échoue :

```bash
echo "nameserver 8.8.8.8" >> /etc/resolv.conf
```

### 4.2 Mettre à jour l'index des sources

```bash
suricata-update update-sources
```

Résultat attendu :
```
Downloading https://www.openinfosecfoundation.org/rules/index.yaml
```

> Si `Warning: Source index does not exist, will use bundled one` apparaît → le DNS n'est pas résolu. Corriger avec le nameserver ci-dessus.

### 4.3 Activer les règles Emerging Threats Open

```bash
suricata-update enable-source et/open
```

Résultat attendu :
```
Source et/open enabled
```

### 4.4 Sources de règles disponibles

```bash
suricata-update list-sources
```

| Source | Type | Description |
|--------|------|-------------|
| `et/open` | Gratuite | Emerging Threats Open — 46 334 règles |
| `et/pro` | Payante | Emerging Threats Pro — couverture étendue |
| `oisf/trafficid` | Gratuite | Identification de protocoles |
| `abuse.ch/feodotracker` | Gratuite | IPs de botnets C2 |
| `abuse.ch/urlhaus` | Gratuite | URLs malveillantes |

### 4.5 Télécharger et déployer les règles

```bash
suricata-update
```

**Résultat attendu :**
```
Fetching https://rules.emergingthreats.net/open/suricata-7.0.10/emerging.rules.tar.gz
Writing rules to /var/lib/suricata/rules/suricata.rules: total: 46334; enabled: 46334
```

> ⚠️ Après `suricata-update`, **toujours redémarrer Suricata** pour charger les règles en mémoire. La RAM passera de ~70 MB (règles de base) à ~750 MB (règles ET/Open chargées).

### 4.6 Valider la configuration

```bash
suricata -T -c /etc/suricata/suricata.yaml -v
```

Résultat attendu :
```
Configuration provided was successfully loaded.
```

### 4.7 Architecture des fichiers

| Fichier | Emplacement |
|---------|-------------|
| Configuration principale | `/etc/suricata/suricata.yaml` |
| Règles consolidées (46 334) | `/var/lib/suricata/rules/suricata.rules` |
| Règles locales custom | `/etc/suricata/rules/` |
| Classification des alertes | `/var/lib/suricata/rules/classification.config` |
| Logs JSON (EVE) | `/var/log/suricata/eve.json` |
| Statistiques | `/var/log/suricata/stats.log` |
| PID | `/var/run/suricata.pid` |
| Cache des sources | `/var/lib/suricata/update/cache/` |

---

## 5. Intégration avec Wazuh

### 5.1 Configurer Wazuh pour lire les logs Suricata

```bash
micro /var/ossec/etc/ossec.conf
```

Ajouter dans `<ossec_config>`, après les blocs `<localfile>` existants :

```xml
<localfile>
  <log_format>json</log_format>
  <location>/var/log/suricata/eve.json</location>
</localfile>
```

### 5.2 Corriger les permissions

```bash
chmod 644 /var/log/suricata/eve.json
# Ne pas modifier l'owner — Suricata doit pouvoir écrire dans ce fichier
# Vérifier : ls -la /var/log/suricata/eve.json
```

Résultat attendu : `-rw-r--r-- 1 suricata suricata`

### 5.3 Résoudre les erreurs de décodage JSON

Si des erreurs apparaissent dans `/var/ossec/logs/ossec.log` :

```
ERROR: Too many fields for JSON decoder
```

Causé par les événements `stats` très volumineux de Suricata. Ces erreurs **n'empêchent pas** la transmission des alertes de sécurité.

```bash
micro /var/ossec/etc/internal_options.conf
# Modifier ou ajouter :
# analysisd.decoder_order_size=512

systemctl restart wazuh-manager
```

### 5.4 Redémarrer et vérifier

```bash
systemctl restart wazuh-manager

# Vérifier que Wazuh lit eve.json
# ⚠️ Utiliser -a si grep signale "fichiers binaires correspondent"
grep -ia "eve.json" /var/ossec/logs/ossec.log
grep -ia "suricata" /var/ossec/logs/ossec.log
```

Résultat attendu : lignes indiquant que le `wazuh-logcollector` analyse bien `/var/log/suricata/eve.json`.

### 5.5 Inspecter les événements JSON

```bash
# Voir les derniers événements bruts
tail -50 /var/log/suricata/eve.json | jq

# Exemple d'événement DNS attendu :
# {
#   "timestamp": "...", "in_iface": "enp0s3",
#   "event_type": "dns", "src_ip": "...", "dest_ip": "...",
#   "dns": { "type": "answer", "rrname": "...", "rrtype": "A" }
# }
```

---

## 6. Démarrage et automatisation

### 6.1 Démarrer Suricata après mise à jour des règles

```bash
# Toujours redémarrer après suricata-update, pas seulement reload
systemctl restart suricata
systemctl status suricata
```

Résultat attendu :
```
Active: active (running) since ...
Main PID: XXXX (Suricata-Main)
Memory: ~750M  ← indique que les règles ET/Open sont bien chargées
```

### 6.2 Activer le démarrage automatique

```bash
systemctl enable suricata
```

### 6.3 Automatiser la mise à jour des règles (cron)

```bash
crontab -e
```

Ajouter :

```cron
0 2 * * * /usr/bin/suricata-update && systemctl restart suricata
```

> Met à jour les règles chaque nuit à 2h00. Utiliser `restart` plutôt que `reload` qui peut expirer avec un grand nombre de règles.

```bash
# Vérifier
crontab -l
```

### 6.4 Rotation des logs

```bash
micro /etc/logrotate.d/suricata
```

```
/var/log/suricata/*.log /var/log/suricata/*.json {
    daily
    rotate 14
    missingok
    compress
    copytruncate
    sharedscripts
    postrotate
        /bin/kill -HUP $(cat /var/run/suricata.pid)
    endscript
}
```

---

## 7. Tests de détection

### 7.1 Générer des événements de test

```bash
# Test 1 : déclencher une règle de détection spécifique
curl http://testmynids.org/uid/index.html
sleep 2

# Test 2 : déclencher la règle GPL ATTACK_RESPONSE
# La réponse retourne uid=0(root) gid=0(root) groups=0(root)
curl http://www.testmyids.com
```

### 7.2 Vérifier les alertes dans eve.json

```bash
# Alertes en temps réel
tail -f /var/log/suricata/eve.json | jq 'select(.event_type == "alert")'

# Résumé des derniers événements
tail -50 /var/log/suricata/eve.json | jq '{
  type: .event_type,
  src: .src_ip,
  dest: .dest_ip,
  alert: .alert.signature
}'
```

Alerte attendue après le test :
```json
{
  "event_type": "alert",
  "src_ip": "...",
  "dest_ip": "...",
  "alert": {
    "signature": "GPL ATTACK_RESPONSE id check returned root",
    "category": "Potentially Bad Traffic",
    "severity": 2
  }
}
```

### 7.3 Vérifier dans Wazuh Dashboard

1. Aller dans **Threat Hunting** → **Events**
2. Filtrer par `data.event_type: alert`
3. Les alertes Suricata apparaissent avec les métadonnées complètes :
   - `alert.signature` — nom de la règle déclenchée
   - `alert.category` — catégorie de la menace
   - `alert.severity` — niveau de sévérité (1 = critique, 4 = info)
   - IP source / destination et ports
   - Interface réseau (`in_iface`)
   - Timestamp précis

---

## 8. Opérations courantes

### État du service

```bash
systemctl status suricata
```

### Logs en temps réel

```bash
# Alertes uniquement
tail -f /var/log/suricata/eve.json | jq 'select(.event_type == "alert")'

# Statistiques
tail -f /var/log/suricata/stats.log
```

### Inspecter les règles déployées

```bash
# Nombre de règles chargées
wc -l /var/lib/suricata/rules/suricata.rules

# Rechercher une règle spécifique
grep -i "ssh" /var/lib/suricata/rules/suricata.rules | head -20
grep -i "attack_response" /var/lib/suricata/rules/suricata.rules | head -5
```

### Mettre à jour les règles manuellement

```bash
suricata-update
systemctl restart suricata
```

### Recharger la configuration sans interruption

```bash
systemctl reload suricata
```

> ⚠️ Le reload peut expirer (`Reload operation timed out`) avec un grand volume de règles. Utiliser `restart` dans ce cas.

---

## 9. Troubleshooting

### `ethtool : commande introuvable`

```bash
apt install -y ethtool
ethtool -K enp0s3 gro off lro off
```

---

### Erreur DNS lors de `suricata-update`

```
Failed to fetch ... <urlopen error [Errno -3] Temporary failure in name resolution>
```

```bash
echo "nameserver 8.8.8.8" >> /etc/resolv.conf
suricata-update update-sources
suricata-update
```

---

### Suricata ne démarre pas

```bash
# Consulter les logs système
journalctl -u suricata -n 50

# Valider la configuration
suricata -T -c /etc/suricata/suricata.yaml -v

# Vérifier les permissions des logs
ls -la /var/log/suricata/
```

---

### Aucune alerte dans eve.json après les tests

**Cause probable 1 — Suricata a démarré avant la mise à jour des règles**

La RAM à ~70 MB indique les règles de base seulement (~428 règles). Les règles ET/Open chargées font passer la RAM à ~750 MB.

```bash
systemctl status suricata
# Vérifier : Memory: ~750M (règles ET/Open) vs ~70M (règles de base)

# Solution : redémarrer après suricata-update
systemctl restart suricata
```

**Cause probable 2 — Mauvaise interface dans la configuration**

```bash
# Vérifier l'interface configurée
grep -A 3 "af-packet" /etc/suricata/suricata.yaml | head -5

# Vérifier que l'interface est UP
ip link show enp0s3
```

**Cause probable 3 — `reload` expiré au lieu de `restart`**

Si `systemctl reload` a expiré, les règles n'ont pas été rechargées. Utiliser `restart`.

```bash
# Tester après correction
curl http://www.testmyids.com
sleep 3
tail -10 /var/log/suricata/eve.json | jq 'select(.event_type == "alert")'
```

---

### Wazuh ne reçoit pas les alertes Suricata

```bash
# 1. Vérifier la configuration localfile dans ossec.conf
grep -A 4 "eve.json" /var/ossec/etc/ossec.conf

# 2. Vérifier les permissions du fichier eve.json
ls -la /var/log/suricata/eve.json
# → doit être lisible par root (644 minimum)

# 3. Vérifier les logs Wazuh
grep -ia "suricata\|eve\|json" /var/ossec/logs/ossec.log

# 4. Corriger les permissions si nécessaire
chmod 644 /var/log/suricata/eve.json
systemctl restart wazuh-manager
```

---

### `grep` retourne "fichiers binaires correspondent"

```bash
# Utiliser l'option -a pour forcer le traitement comme texte
grep -ia "terme" /var/ossec/logs/ossec.log
```

---

### Erreurs de décodage JSON dans Wazuh

```
ERROR: Too many fields for JSON decoder
```

Causé par les événements `stats` volumineux. Normal, n'affecte pas les alertes.

```bash
micro /var/ossec/etc/internal_options.conf
# analysisd.decoder_order_size=512
systemctl restart wazuh-manager
```

---

### `systemctl reload` expire (Reload operation timed out)

Normal avec un grand nombre de règles. Utiliser `restart` :

```bash
systemctl restart suricata
```

---

### Performances dégradées (CPU élevé)

```bash
top -p $(pgrep suricata)
cat /var/log/suricata/stats.log | tail -50

# Augmenter les threads AF_PACKET
micro /etc/suricata/suricata.yaml
# → af-packet: threads: 4  (selon nombre de cœurs disponibles)

systemctl restart suricata
```

---

### Faux positifs — Désactiver une règle

```bash
# Trouver le SID de la règle dans les alertes
tail -f /var/log/suricata/eve.json | jq 'select(.event_type == "alert") | .alert'

# Désactiver le SID (ex: SID 2100498)
echo "disable-rule 2100498" >> /etc/suricata/disable.conf
suricata-update
systemctl restart suricata
```

---

## Références

| Ressource | URL |
|-----------|-----|
| Documentation Suricata | https://suricata.io/docs/ |
| Wiki Emerging Threats | https://doc.emergingthreats.net/ |
| Documentation Wazuh | https://documentation.wazuh.com/ |
| Forum Suricata | https://forum.suricata.io/ |
