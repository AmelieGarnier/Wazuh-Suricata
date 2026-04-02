# Runbook Wazuh — SIEM

> **Environnement :** Debian 13 — IP serveur : `<IP-SERVEUR>`
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
| OS | Debian 13 64-bit |
| RAM | 8 GB |
| CPU | 4 cœurs |
| Disque | 100 GB |
| Réseau | Mode pont (Bridged) — obligatoire |

---

## Table des matières

1. [Prérequis](#1-prérequis)
2. [Installation](#2-installation)
3. [Configuration des composants](#3-configuration-des-composants)
4. [Intégration des agents](#4-intégration-des-agents)
5. [Configuration FIM](#5-configuration-fim)
6. [Active Response — Brute Force](#6-active-response--brute-force)
7. [Intégration VirusTotal](#7-intégration-virustotal)
8. [Opérations courantes](#8-opérations-courantes)
9. [Troubleshooting](#9-troubleshooting)

---

## 1. Prérequis

Avant de commencer, s'assurer que :

- Accès `root` sur la machine
- Connexion Internet stable
- Réseau VirtualBox en mode **Accès par pont** (pas NAT)
- Ports disponibles : `443` (Dashboard), `9200` (Indexer), `1514-1515` (Manager), `55000` (API Wazuh Manager)
- Mises à jour système appliquées

> Toutes les commandes sont exécutées en tant que `root`.

### Passer root

```bash
sudo -i
```

### Corriger le PATH (commandes sbin manquantes)

Sur certaines installations minimales, `/sbin` et `/usr/sbin` ne sont pas dans le PATH. À faire en premier :

```bash
export PATH=$PATH:/sbin:/usr/sbin
```

> **Symptôme si absent** : `bash: sysctl : commande introuvable` ou `bash: runuser : commande introuvable`
> Pour rendre permanent : `echo 'export PATH=$PATH:/sbin:/usr/sbin' >> /root/.bashrc && source /root/.bashrc`

### Récupérer l'adresse IP

```bash
ip a | grep "inet " | grep -v 127
```

> Note cette IP — elle remplace `<IP-SERVEUR>` dans toutes les commandes suivantes.

### Paramètre noyau requis (OpenSearch)

OpenSearch (Wazuh Indexer) requiert un `vm.max_map_count` élevé. Sans ce paramètre, le service refuse de démarrer.

```bash
apt install -y procps
sysctl -w vm.max_map_count=262144
echo "vm.max_map_count=262144" >> /etc/sysctl.conf
```

> **Si `sysctl` reste introuvable** après installation de `procps` : utiliser `/sbin/sysctl -w vm.max_map_count=262144`

### Outils système

```bash
apt install -y curl gpg micro
```

> Si `micro` échoue à cause d'un problème DNS temporaire, réessayer après correction du DNS (voir Troubleshooting).

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
# Télécharger les outils — version 4.13
curl -sO https://packages.wazuh.com/4.13/wazuh-certs-tool.sh
curl -sO https://packages.wazuh.com/4.13/config.yml
```

Vérifier que les fichiers sont présents :

```bash
ls -lh wazuh-certs-tool.sh config.yml
```

Éditer `config.yml` avec l'IP réelle du serveur :

```bash
micro config.yml
```

```yaml
nodes:
  indexer:
    - name: node-1
      ip: <IP-SERVEUR>
  server:
    - name: wazuh-1
      ip: <IP-SERVEUR>
  dashboard:
    - name: dashboard
      ip: <IP-SERVEUR>
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
apt install -y wazuh-indexer wazuh-manager wazuh-dashboard filebeat
```

> **Note** : `filebeat` doit être installé ici en même temps que les autres paquets, tant que le dépôt Wazuh est actif.

### Désactiver le dépôt après installation

Pour éviter une mise à jour automatique non maîtrisée :

```bash
sed -i "s/^deb/#deb/" /etc/apt/sources.list.d/wazuh.list && apt-get update
```

> Pour réactiver lors d'une mise à jour intentionnelle : `sed -i "s/^#deb/deb/" /etc/apt/sources.list.d/wazuh.list && apt-get update`

---

## 3. Configuration des composants

### 3.1 Wazuh Indexer

```bash
# Configurer l'adresse réseau
micro /etc/wazuh-indexer/opensearch.yml
# → network.host: "<IP-SERVEUR>"

# Définir la variable NODE_NAME (doit correspondre au nom dans config.yml)
NODE_NAME=node-1

# Déployer les certificats
mkdir /etc/wazuh-indexer/certs
tar -xf ./wazuh-certificates.tar -C /etc/wazuh-indexer/certs/ \
  ./$NODE_NAME.pem ./$NODE_NAME-key.pem ./admin.pem ./admin-key.pem ./root-ca.pem

mv /etc/wazuh-indexer/certs/$NODE_NAME.pem     /etc/wazuh-indexer/certs/indexer.pem
mv /etc/wazuh-indexer/certs/$NODE_NAME-key.pem /etc/wazuh-indexer/certs/indexer-key.pem
chmod 500 /etc/wazuh-indexer/certs
chmod 400 /etc/wazuh-indexer/certs/*
chown -R wazuh-indexer:wazuh-indexer /etc/wazuh-indexer/certs

# Démarrer
systemctl daemon-reload
systemctl enable wazuh-indexer
systemctl start wazuh-indexer

# Corriger le PATH avant d'initialiser le cluster
export PATH=$PATH:/sbin:/usr/sbin

# Initialiser le cluster de sécurité
/usr/share/wazuh-indexer/bin/indexer-security-init.sh

# Tester
curl -k -u admin:admin https://<IP-SERVEUR>:9200
curl -k -u admin:admin https://<IP-SERVEUR>:9200/_cat/nodes?v
```

> **Si `runuser : commande introuvable`** lors de `indexer-security-init.sh` : vérifier que `export PATH=$PATH:/sbin:/usr/sbin` a bien été exécuté.

### 3.2 Wazuh Manager

```bash
systemctl daemon-reload
systemctl enable wazuh-manager
systemctl start wazuh-manager
systemctl status wazuh-manager
```

### 3.3 Filebeat

```bash
# Télécharger la configuration — version 4.13
curl -so /etc/filebeat/filebeat.yml \
  https://packages.wazuh.com/4.13/tpl/wazuh/filebeat/filebeat.yml

# Configurer l'IP de l'Indexer dans filebeat.yml
micro /etc/filebeat/filebeat.yml
# → output.elasticsearch.hosts: ["<IP-SERVEUR>:9200"]

# Configurer le keystore
filebeat keystore create
echo admin | filebeat keystore add username --stdin --force
echo admin | filebeat keystore add password --stdin --force
# ⚠️ Note de sécurité : en production, remplacer admin/admin par des identifiants personnalisés

# Installer le module Wazuh — version 0.5
curl -s https://packages.wazuh.com/4.x/filebeat/wazuh-filebeat-0.5.tar.gz \
  | tar -xvz -C /usr/share/filebeat/module

# Télécharger le template d'index
curl -so /etc/filebeat/wazuh-template.json \
  https://raw.githubusercontent.com/wazuh/wazuh/v4.13.0/extensions/elasticsearch/7.x/wazuh-template.json
chmod go+r /etc/filebeat/wazuh-template.json

# Certificats Filebeat
NODE_NAME=wazuh-1

mkdir /etc/filebeat/certs
tar -xf ./wazuh-certificates.tar -C /etc/filebeat/certs/ \
  ./$NODE_NAME.pem ./$NODE_NAME-key.pem ./root-ca.pem
mv /etc/filebeat/certs/$NODE_NAME.pem     /etc/filebeat/certs/filebeat.pem
mv /etc/filebeat/certs/$NODE_NAME-key.pem /etc/filebeat/certs/filebeat-key.pem
chmod 500 /etc/filebeat/certs
chmod 400 /etc/filebeat/certs/*
chown -R root:root /etc/filebeat/certs

# Charger le template d'index dans l'Indexer
filebeat setup --index-management \
  -E output.logstash.enabled=false \
  -E "output.elasticsearch.hosts=['https://<IP-SERVEUR>:9200']" \
  -E "output.elasticsearch.username=admin" \
  -E "output.elasticsearch.password=admin" \
  -E "output.elasticsearch.ssl.verification_mode=none"

# Résultat attendu : "Index setup finished."
# L'erreur Kibana/port 5601 est normale — ignorer

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
# → opensearch.hosts: ["https://<IP-SERVEUR>:9200"]
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

**Accès web :** `https://<IP-SERVEUR>` — identifiants par défaut : `admin / admin`

> ⚠️ **Changer le mot de passe admin** après la première connexion via le Dashboard → Administration → Security → Users.

### 3.5 Configurer la connexion Dashboard → Manager (API)

```bash
micro /usr/share/wazuh-dashboard/data/wazuh/config/wazuh.yml
```

```yaml
hosts:
  - default:
      url: https://<IP-SERVEUR>
      port: 55000
      username: wazuh-wui
      password: wazuh-wui
      run_as: true
```

```bash
systemctl restart wazuh-dashboard
```

---

## 4. Intégration des agents

### 4.1 Agent Linux (Debian 13)

```bash
# Sur la machine agent — réactiver le dépôt si nécessaire
wget https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.13.1-1_amd64.deb \
  && WAZUH_MANAGER='<IP-SERVEUR>' \
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
  WAZUH_MANAGER='<IP-SERVEUR>' `
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
    <host>https://<IP-SERVEUR>:9200</host>
  </hosts>
  <ssl>
    <certificate_authorities>
      <ca>/etc/wazuh-indexer/certs/root-ca.pem</ca>
    </certificate_authorities>
    <certificate>/etc/filebeat/certs/filebeat.pem</certificate>
    <key>/etc/filebeat/certs/filebeat-key.pem</key>
  </ssl>
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

## 7. Intégration VirusTotal

Wazuh peut enrichir automatiquement les alertes FIM en soumettant les hash de fichiers suspects à l'API VirusTotal.

### 7.1 Récupérer la clé API VirusTotal

1. Se connecter sur [https://www.virustotal.com](https://www.virustotal.com)
2. Cliquer sur l'**icône de profil** en haut à droite
3. Ouvrir **API Keys**
4. Copier la clé affichée

> ⚠️ **Sécurité — ne jamais :**
> - Committer la clé dans un dépôt Git (même privé)
> - La laisser apparaître dans des captures d'écran
> - La laisser dans l'historique shell (`history -c` pour effacer)
>
> Si la clé a été exposée, la régénérer immédiatement depuis l'interface VirusTotal.

### 7.2 Configurer l'intégration dans Wazuh

```bash
micro /var/ossec/etc/ossec.conf
```

Ajouter dans la section `<ossec_config>` :

```xml
<integration>
  <n>virustotal</n>
  <api_key>VOTRE_CLE_API_ICI</api_key>
  <group>syscheck</group>
  <alert_format>json</alert_format>
</integration>
```

> Remplacer `VOTRE_CLE_API_ICI` par la valeur copiée depuis VirusTotal. Ne pas entourer la clé de chevrons `< >`.

```bash
systemctl restart wazuh-manager
```

---

## 8. Opérations courantes

### Vérifier l'état de tous les services

```bash
systemctl status wazuh-manager wazuh-indexer wazuh-dashboard filebeat
```

### Logs en temps réel

```bash
# Manager
tail -f /var/ossec/logs/ossec.log

# Indexer
journalctl -u wazuh-indexer -f

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

## 9. Troubleshooting

### `sysctl : commande introuvable` ou `runuser : commande introuvable`

```bash
# Corriger le PATH
export PATH=$PATH:/sbin:/usr/sbin

# Rendre permanent
echo 'export PATH=$PATH:/sbin:/usr/sbin' >> /root/.bashrc
source /root/.bashrc
```

---

### Erreur permissions keyring GPG

```
Sub-process /usr/bin/sqv returned an error code (1)
Permission denied reading "/usr/share/keyrings/wazuh.gpg"
```

```bash
chmod 644 /usr/share/keyrings/wazuh.gpg
apt update
```

---

### Erreur DNS temporaire lors de `apt update`

```
Erreur temporaire de résolution de « deb.debian.org »
```

```bash
echo "nameserver 8.8.8.8" >> /etc/resolv.conf
apt update
```

---

### `Impossible de trouver le paquet filebeat`

Le dépôt Wazuh a été désactivé avant l'installation de filebeat :

```bash
sed -i "s/^#deb/deb/" /etc/apt/sources.list.d/wazuh.list
apt update
apt install -y filebeat
sed -i "s/^deb/#deb/" /etc/apt/sources.list.d/wazuh.list
apt update
```

---

### Erreur : `Connection refused` sur le port 9200 (Indexer)

```bash
# 1. Vérifier le service
systemctl status wazuh-indexer

# 2. Vérifier via journalctl (le fichier de log peut ne pas exister)
journalctl -u wazuh-indexer -n 50 --no-pager

# 3. Vérifier vm.max_map_count (cause la plus fréquente)
sysctl vm.max_map_count
# → doit retourner 262144. Si inférieur :
sysctl -w vm.max_map_count=262144

# 4. Vérifier l'écoute réseau
ss -tulpn | grep 9200

# 5. Tester avec l'IP réelle (pas 127.0.0.1)
curl -k -u admin:admin https://<IP-SERVEUR>:9200

# 6. Autoriser le port si firewall actif
ufw allow 9200/tcp
```

---

### Erreur Dashboard : "no template found for wazuh-alerts-*"

Le template d'index Filebeat n'est pas chargé :

```bash
# Vérifier que le template existe
ls -lh /etc/filebeat/wazuh-template.json

# Si absent, le télécharger
curl -so /etc/filebeat/wazuh-template.json \
  https://raw.githubusercontent.com/wazuh/wazuh/v4.13.0/extensions/elasticsearch/7.x/wazuh-template.json
chmod go+r /etc/filebeat/wazuh-template.json

# Charger le template
filebeat setup --index-management \
  -E output.logstash.enabled=false \
  -E "output.elasticsearch.hosts=['https://<IP-SERVEUR>:9200']" \
  -E "output.elasticsearch.username=admin" \
  -E "output.elasticsearch.password=admin" \
  -E "output.elasticsearch.ssl.verification_mode=none"

# Résultat attendu : "Index setup finished."

systemctl restart filebeat wazuh-manager wazuh-indexer wazuh-dashboard
```

---

### Agent `Never connected` ou `Disconnected`

```bash
# Sur l'AGENT — vérifier la configuration du Manager
cat /var/ossec/etc/ossec.conf | grep -A 5 "<client>"

# Tester la connectivité (port Manager)
telnet <IP-SERVEUR> 1514

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
