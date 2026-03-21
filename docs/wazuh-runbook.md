# Runbook Wazuh — SIEM

> **Environnement :** Debian 12 — IP serveur : `192.168.111.62`
> **Version Wazuh :** 4.13.x

---

## Présentation de Wazuh

Wazuh est une solution open-source de SIEM (Security Information and Event Management) qui permet de surveiller l'infrastructure en temps réel, détecter les menaces et répondre aux incidents. Elle excelle dans les domaines suivants :

- **Détection des intrusions (IDS)** — Analyse comportementale et détection d'anomalies
- **Surveillance de l'intégrité des fichiers (FIM)** — Détection de modifications non autorisées
- **Réponse aux incidents** — Automatisation des actions de remédiation (Active Response)
- **Conformité réglementaire** — PCI-DSS, GDPR, HIPAA, CIS
- **Analyse de vulnérabilités** — Identification des failles de sécurité
- **Détection de malwares** — Intégration avec VirusTotal

### Architecture all-in-one (single-node)

Tous les composants sont installés sur un seul serveur. Cette configuration est adaptée aux environnements de test, petites infrastructures (< 100 agents) et POC.

| Composant | Rôle |
|-----------|------|
| **Wazuh Indexer** (OpenSearch) | Stockage, indexation et recherche des données |
| **Wazuh Manager** | Cœur du système — analyse, corrélation, alertes |
| **Filebeat** | Collecte et transmission des logs vers l'Indexer |
| **Wazuh Dashboard** | Interface web de visualisation et gestion |

> Pour les environnements de production à grande échelle, une architecture **multi-node** avec serveurs dédiés est recommandée.

### Spécifications de la VM

| Ressource | Valeur |
|-----------|--------|
| OS | Debian 12 (Bookworm) 64-bit |
| RAM | 8 GB |
| CPU | 4 cœurs |
| Disque | 100 GB |

---

## Table des matières

1. [Prérequis](#1-prérequis)
2. [Installation](#2-installation)
3. [Configuration des composants](#3-configuration-des-composants)
4. [Intégration des agents](#4-intégration-des-agents)
5. [Configuration FIM](#5-configuration-fim)
6. [Active Response — Brute Force](#6-active-response--brute-force)
7. [Opérations courantes](#7-opérations-courantes)
8. [Troubleshooting](#8-troubleshooting)

---

## 1. Prérequis

Avant de commencer, s'assurer que :

- Accès `root` sur la machine Debian 12
- Connexion Internet stable
- Ports disponibles : `443` (Dashboard), `9200` (Indexer), `1514-1515` (Manager)
- Mises à jour système appliquées

> Toutes les commandes sont exécutées en tant que `root` — pas besoin de `sudo`.

### Outils système

```bash
apt install -y curl gpg micro
```

### Récupérer l'adresse IP

```bash
ip a
```

### Ajouter le dépôt Wazuh

```bash
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH \
  | gpg --no-default-keyring --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg --import \
  && chmod 644 /usr/share/keyrings/wazuh.gpg

echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" \
  | tee -a /etc/apt/sources.list.d/wazuh.list
```

### Mettre à jour le système

```bash
apt update && apt upgrade -y
apt-get install -y debconf adduser procps curl gnupg apt-transport-https debhelper libcap2-bin
```

---

## 2. Installation

### Générer les certificats SSL/TLS

```bash
# Télécharger les outils
curl -sO https://packages.wazuh.com/4.8/wazuh-certs-tool.sh
curl -sO https://packages.wazuh.com/4.8/config.yml
```

Éditer `config.yml` avec les noms et IPs de vos nœuds :

```bash
micro config.yml
```

```yaml
nodes:
  indexer:
    - name: node-1
      ip: 192.168.111.62
  server:
    - name: wazuh-1
      ip: 192.168.111.62
  dashboard:
    - name: dashboard
      ip: 192.168.111.62
```

```bash
# Générer les certificats
bash ./wazuh-certs-tool.sh -A

# Compresser
tar -cvf ./wazuh-certificates.tar -C ./wazuh-certificates/ .
rm -rf ./wazuh-certificates
```

### Installer les paquets

```bash
apt install -y wazuh-indexer wazuh-manager wazuh-dashboard
```

---

## 3. Configuration des composants

### 3.1 Wazuh Indexer

```bash
# Configurer l'adresse réseau
micro /etc/wazuh-indexer/opensearch.yml
# → network.host: "192.168.111.62"
# Alternative pour écouter sur toutes les interfaces : network.host: "0.0.0.0"

# Définir la variable NODE_NAME (doit correspondre au nom dans config.yml)
NODE_NAME=node-1

# Déployer les certificats
mkdir /etc/wazuh-indexer/certs
tar -xf ./wazuh-certificates.tar -C /etc/wazuh-indexer/certs/ \
  ./$NODE_NAME.pem ./$NODE_NAME-key.pem ./admin.pem ./admin-key.pem ./root-ca.pem

# Erreur fréquente : "fichier non trouvé" → vérifier :
# 1. echo $NODE_NAME  (variable bien définie)
# 2. ls -lh wazuh-certificates.tar  (archive présente)

mv /etc/wazuh-indexer/certs/$NODE_NAME.pem     /etc/wazuh-indexer/certs/indexer.pem
mv /etc/wazuh-indexer/certs/$NODE_NAME-key.pem /etc/wazuh-indexer/certs/indexer-key.pem
chmod 500 /etc/wazuh-indexer/certs
chmod 400 /etc/wazuh-indexer/certs/*
chown -R wazuh-indexer:wazuh-indexer /etc/wazuh-indexer/certs

# Démarrer
systemctl daemon-reload
systemctl enable wazuh-indexer
systemctl start wazuh-indexer

# Initialiser le cluster de sécurité
/usr/share/wazuh-indexer/bin/indexer-security-init.sh

# Tester
curl -k -u admin:admin https://192.168.111.62:9200
curl -k -u admin:admin https://192.168.111.62:9200/_cat/nodes?v
```

### 3.2 Wazuh Manager

```bash
systemctl daemon-reload
systemctl enable wazuh-manager
systemctl start wazuh-manager
systemctl status wazuh-manager
```

### 3.3 Filebeat

```bash
apt install -y filebeat

# Télécharger la configuration
curl -so /etc/filebeat/filebeat.yml \
  https://packages.wazuh.com/4.8/tpl/wazuh/filebeat/filebeat.yml

# Configurer l'IP de l'Indexer dans filebeat.yml
micro /etc/filebeat/filebeat.yml
# → output.elasticsearch.hosts: ["192.168.111.62:9200"]

# Configurer le keystore
filebeat keystore create
echo admin | filebeat keystore add username --stdin --force
echo admin | filebeat keystore add password --stdin --force
# ⚠️ Note de sécurité : en production, remplacer admin/admin par des identifiants personnalisés

# Installer le template et le module Wazuh
curl -so /etc/filebeat/wazuh-template.json \
  https://raw.githubusercontent.com/wazuh/wazuh/v4.8.2/extensions/elasticsearch/7.x/wazuh-template.json
chmod go+r /etc/filebeat/wazuh-template.json

curl -s https://packages.wazuh.com/4.x/filebeat/wazuh-filebeat-0.4.tar.gz \
  | tar -xvz -C /usr/share/filebeat/module

# Certificats Filebeat
# Définir le nom du nœud serveur (doit correspondre au config.yml)
NODE_NAME=wazuh-1

mkdir /etc/filebeat/certs
tar -xf ./wazuh-certificates.tar -C /etc/filebeat/certs/ \
  ./$NODE_NAME.pem ./$NODE_NAME-key.pem ./root-ca.pem
mv /etc/filebeat/certs/$NODE_NAME.pem     /etc/filebeat/certs/filebeat.pem
mv /etc/filebeat/certs/$NODE_NAME-key.pem /etc/filebeat/certs/filebeat-key.pem
chmod 500 /etc/filebeat/certs
chmod 400 /etc/filebeat/certs/*
chown -R root:root /etc/filebeat/certs

# Démarrer
systemctl daemon-reload
systemctl enable filebeat
systemctl start filebeat

# Tester
filebeat test output
```

### 3.4 Wazuh Dashboard

```bash
# Configurer l'IP OpenSearch
micro /etc/wazuh-dashboard/opensearch_dashboards.yml
# → opensearch.hosts: ["https://192.168.111.62:9200"]
# → server.host: "0.0.0.0"

# Certificats Dashboard
mkdir /etc/wazuh-dashboard/certs
tar -xf ./wazuh-certificates.tar -C /etc/wazuh-dashboard/certs/ \
  ./dashboard.pem ./dashboard-key.pem ./root-ca.pem
chmod 500 /etc/wazuh-dashboard/certs
chmod 400 /etc/wazuh-dashboard/certs/*
chown -R wazuh-dashboard:wazuh-dashboard /etc/wazuh-dashboard/certs

# Démarrer
systemctl daemon-reload
systemctl enable wazuh-dashboard
systemctl start wazuh-dashboard

# Vérifier les 3 services
systemctl status wazuh-manager wazuh-indexer wazuh-dashboard
```

**Accès web :** `https://192.168.111.62` — identifiants par défaut : `admin / admin`

---

## 4. Intégration des agents

### 4.1 Agent Linux (Debian)

```bash
# Sur la machine agent
wget https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.13.1-1_amd64.deb \
  && WAZUH_MANAGER='192.168.111.62' \
     WAZUH_AGENT_GROUP='default' \
     WAZUH_AGENT_NAME='<nom-agent>' \
     dpkg -i ./wazuh-agent_4.13.1-1_amd64.deb

systemctl daemon-reload
systemctl enable wazuh-agent
systemctl start wazuh-agent
systemctl status wazuh-agent
```

### 4.2 Agent Windows 10 (PowerShell en Admin)

```powershell
Invoke-WebRequest -Uri https://packages.wazuh.com/4.x/windows/wazuh-agent-4.13.1-1.msi `
  -OutFile $env:tmp\wazuh-agent.msi

msiexec.exe /i $env:tmp\wazuh-agent.msi /q `
  WAZUH_MANAGER='192.168.111.62' `
  WAZUH_AGENT_GROUP='default' `
  WAZUH_AGENT_NAME='<nom-agent>'

NET START Wazuh
```

### 4.3 Vérifier les agents depuis le Manager

```bash
# Lister les agents
/var/ossec/bin/agent_control -l

# Détails d'un agent
/var/ossec/bin/agent_control -i 001
```

---

## 5. Configuration FIM

Le **File Integrity Monitoring (FIM)** surveille les modifications de fichiers et répertoires en temps réel.

### 5.1 FIM sur Windows

Fichier de configuration : `C:\Program Files (x86)\ossec-agent\ossec.conf`

```xml
<syscheck>
  <directories check_all="yes" realtime="yes" report_changes="yes">
    C:\Users\<utilisateur>\Documents
  </directories>
</syscheck>
```

Redémarrer l'agent via `services.msc` → service **Wazuh**.

### 5.2 FIM sur Linux

Fichier de configuration : `/var/ossec/etc/ossec.conf`

```xml
<syscheck>
  <directories check_all="yes" realtime="yes" report_changes="yes">
    /etc,/usr/bin,/usr/sbin
  </directories>
</syscheck>
```

```bash
systemctl restart wazuh-agent
systemctl status wazuh-agent
```

### 5.3 Mode Whodata (audit avancé)

Le mode **Whodata** enregistre quel utilisateur / processus a modifié un fichier.

**Windows :**

```xml
<syscheck>
  <directories check_all="yes" whodata="yes">
    C:\Users\<utilisateur>\Documents
  </directories>
</syscheck>
```

Redémarrer le service Wazuh.

**Linux — prérequis :**

```bash
apt-get install -y auditd audispd-plugins
systemctl restart auditd
```

```xml
<syscheck>
  <directories check_all="yes" whodata="yes">
    /etc
  </directories>
</syscheck>
```

```bash
systemctl restart wazuh-agent
```

### 5.4 Résolution du problème de connexion à l'Indexer

Si le Dashboard affiche une erreur de connexion à l'Indexer :

```bash
micro /var/ossec/etc/ossec.conf
```

```xml
<indexer>
  <enabled>yes</enabled>
  <hosts>
    <host>https://192.168.111.62:9200</host>
  </hosts>
  ...
</indexer>
```

```bash
systemctl restart wazuh-manager
systemctl status wazuh-manager
```

---

## 6. Active Response — Brute Force

L'**Active Response** bloque automatiquement les IPs qui génèrent trop d'échecs d'authentification SSH.

### 6.1 Architecture

```
Machine attaquante → SSH → Machine victime (agent) → Wazuh Manager → firewall-drop → Blocage IP
```

### 6.2 Configuration sur le Manager

```bash
micro /var/ossec/etc/ossec.conf
```

Ajouter dans la section `<ossec_config>` :

```xml
<active-response>
  <command>firewall-drop</command>
  <location>local</location>
  <rules_id>5763</rules_id>
  <timeout>600</timeout>
</active-response>
```

> **Règle 5763** : détection d'attaque par force brute SSH (seuil : 8 tentatives)
> **Timeout** : 600 secondes (10 min) avant déblocage automatique

```bash
systemctl restart wazuh-manager
```

### 6.3 Vérifier le blocage

Depuis la machine victime (agent) :

```bash
# Vérifier les règles iptables
iptables -L -n -v | grep <IP-attaquante>

# Logs Active Response
tail -f /var/ossec/logs/active-responses.log

# Tester manuellement
/var/ossec/active-response/bin/firewall-drop add - <IP>
/var/ossec/active-response/bin/firewall-drop delete - <IP>
```

---

## 7. Opérations courantes

### Vérifier l'état de tous les services

```bash
systemctl status wazuh-manager wazuh-indexer wazuh-dashboard filebeat
```

### Logs en temps réel

```bash
# Manager
tail -f /var/ossec/logs/ossec.log

# Indexer
tail -f /var/log/wazuh-indexer/wazuh-indexer.log

# Dashboard
tail -f /var/log/wazuh-dashboard/wazuh-dashboard.log
```

### Gestion des agents

```bash
# Lister tous les agents
/var/ossec/bin/agent_control -l

# Détails d'un agent
/var/ossec/bin/agent_control -i <ID>

# Redémarrer tous les agents depuis le Manager
/var/ossec/bin/agent_control -R -a
```

### Tester des règles

```bash
/var/ossec/bin/wazuh-logtest
```

### Redémarrer tous les services

```bash
systemctl restart wazuh-manager wazuh-indexer wazuh-dashboard filebeat
```

---

## 8. Troubleshooting

### Erreur : `Connection refused` sur le port 9200 (Indexer)

```
curl: (7) Failed to connect to 192.168.111.62 port 9200: Connection refused
```

```bash
# 1. Vérifier le service
systemctl status wazuh-indexer

# 2. Vérifier l'écoute réseau
ss -tulpn | grep 9200

# 3. Vérifier la configuration
cat /etc/wazuh-indexer/opensearch.yml | grep network.host

# 4. Consulter les logs
tail -50 /var/log/wazuh-indexer/wazuh-indexer.log

# 5. Autoriser le port si firewall actif
ufw allow 9200/tcp
```

---

### Agent `Never connected` ou `Disconnected`

```bash
# Sur l'AGENT — vérifier la configuration du Manager
cat /var/ossec/etc/ossec.conf | grep -A 5 "<client>"

# Tester la connectivité (port Manager)
telnet 192.168.111.62 1514

# Consulter les logs de l'agent
tail -f /var/ossec/logs/ossec.log

# Redémarrer l'agent
systemctl restart wazuh-agent

# Sur le MANAGER — vérifier que les ports écoutent
ss -tulpn | grep -E '1514|1515'

# Lister les agents enregistrés
/var/ossec/bin/agent_control -l
```

---

### Certificats SSL invalides

```
ERROR: Certificate verify failed
```

```bash
# Vérifier la validité d'un certificat
openssl x509 -in /etc/wazuh-indexer/certs/indexer.pem -text -noout

# Vérifier la correspondance certificat / clé (les deux hash MD5 doivent être identiques)
openssl x509 -noout -modulus -in /etc/wazuh-indexer/certs/indexer.pem | openssl md5
openssl rsa  -noout -modulus -in /etc/wazuh-indexer/certs/indexer-key.pem | openssl md5

# Recréer les certificats si nécessaire
rm -f wazuh-certificates.tar
bash ./wazuh-certs-tool.sh -A
```

---

### Dashboard inaccessible (`ERR_CONNECTION_REFUSED`)

```bash
# 1. Vérifier le service
systemctl status wazuh-dashboard

# 2. Vérifier l'écoute sur le port 443
ss -tulpn | grep :443

# 3. Vérifier la configuration
cat /etc/wazuh-dashboard/opensearch_dashboards.yml | grep server.host

# 4. Consulter les logs
tail -50 /var/log/wazuh-dashboard/wazuh-dashboard.log

# 5. Redémarrer
systemctl restart wazuh-dashboard
```

---

### Active Response ne bloque pas l'IP

```bash
# Sur la machine VICTIME (agent)
# 1. Vérifier les règles iptables
iptables -L -n -v | grep <IP-attaquante>

# 2. Consulter les logs Active Response
tail -f /var/ossec/logs/active-responses.log

# 3. Tester manuellement le script
/var/ossec/active-response/bin/firewall-drop add - <IP-attaquante>

# 4. Vérifier le blocage
iptables -L -n | grep <IP-attaquante>

# 5. Débloquer manuellement
/var/ossec/active-response/bin/firewall-drop delete - <IP-attaquante>
```
