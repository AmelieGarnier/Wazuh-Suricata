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

#### Test FIM Windows

Créer, modifier ou supprimer un fichier dans le répertoire surveillé (`C:\Users\<utilisateur>\Documents\`), puis vérifier dans le Dashboard → **File Integrity Monitoring** → **Recent events**.

Chaque événement détaille :

| Champ | Description |
|-------|-------------|
| Path | Chemin complet du fichier |
| Action | `added` / `modified` / `deleted` |
| Date | Horodatage de la modification |
| MD5 / SHA1 / SHA256 | Empreintes cryptographiques du fichier |

> Les attributs surveillés avec `check_all="yes"` incluent : taille, propriétaire, permissions, MD5, SHA1, SHA256.

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

#### Test FIM Linux

Créer un fichier dans le répertoire surveillé :

```bash
micro /home/<utilisateur>/Bureau/test-fim.txt
```

Vérifier dans le Dashboard → **File Integrity Monitoring** → **Recent events** :

| Champ | Description |
|-------|-------------|
| Path | Chemin complet du fichier |
| Action | `added` / `modified` / `deleted` |
| MD5 / SHA1 / SHA256 | Empreintes cryptographiques |
| Permissions | Mode du fichier (ex: `rw-r--r--`) |

> Répertoires critiques recommandés à surveiller : `/etc`, `/bin`, `/sbin`, `/usr/bin`, `/home`, `/var/www`

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

#### Test Whodata Windows

Créer un fichier texte dans le répertoire surveillé (ex : Bureau), puis vérifier dans le Dashboard → **File Integrity Monitoring** → **Events**.

Informations supplémentaires disponibles en mode Whodata :

| Champ Dashboard | Description |
|-----------------|-------------|
| `syscheck.audit.user.name` | Nom d'utilisateur Windows (ex: ERIS-FAD) |
| `syscheck.audit.user.id` | SID de l'utilisateur |
| `syscheck.audit.process.name` | Processus ayant effectué la modification (ex: `notepad.exe`) |
| `syscheck.audit.process.id` | PID du processus |
| `syscheck.mode` | `whodata` |

> Ces informations permettent de reconstituer précisément le contexte d'une modification — essentiel lors d'une investigation de sécurité.

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

#### Test Whodata Linux

Créer un fichier dans le répertoire surveillé :

```bash
micro ~/text-who.txt
```

Vérifier dans le Dashboard → **File Integrity Monitoring** → **Events** :

| Champ Dashboard | Description |
|-----------------|-------------|
| `syscheck.audit.effective_user.name` | Utilisateur effectif (ex: `root`) |
| `syscheck.audit.login_user.name` | Utilisateur connecté (ex: `debian-wazu`) |
| `syscheck.audit.process.name` | Commande exécutée (ex: `/usr/bin/micro`) |
| `syscheck.audit.process.id` | PID |
| `syscheck.audit.process.parent_name` | Processus parent (ex: `/usr/bin/bash`) |
| `syscheck.audit.process.ppid` | PPID |
| `syscheck.audit.group.name` | Groupe (ex: `root`) |
| `syscheck.mode` | `whodata` |

> Traçabilité complète requise pour la conformité PCI-DSS, HIPAA et les audits de sécurité.

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

> ⚠️ **Cette démonstration doit être réalisée uniquement dans un environnement de test contrôlé.** L'utilisation d'outils comme Hydra sur des systèmes sans autorisation est illégale.

### 6.1 Architecture du laboratoire

```
Machine attaquante        Machine victime           Wazuh Manager
(debian-siem)             (debian-wazu2)            (192.168.x.x)
192.168.x.61              192.168.x.63
    │                          │                          │
    │── Hydra SSH ────────────►│── logs port 1514 ───────►│
    │                          │                          │── Règle 5763 détectée
    │◄─── IP bloquée (iptables)│◄─── firewall-drop ───────│
```

| Machine | Rôle | Outils |
|---------|------|--------|
| Attaquante | Lance l'attaque brute-force SSH | Hydra, pwgen |
| Victime | Cible SSH avec agent Wazuh | SSH server, agent Wazuh |
| Manager | Détecte et déclenche la réponse | Wazuh Manager |

### 6.2 Configuration de l'Active Response (sur le Manager)

```bash
# Vérifier que la commande firewall-drop est présente
cat /var/ossec/etc/ossec.conf | grep -A 4 "firewall-drop"
```

```bash
micro /var/ossec/etc/ossec.conf
```

Ajouter dans la section `<ossec_config>` :

```xml
<active-response>
  <command>firewall-drop</command>
  <location>local</location>
  <rules_id>5763</rules_id>
  <timeout>180</timeout>
</active-response>
```

| Paramètre | Valeur | Description |
|-----------|--------|-------------|
| `<command>` | firewall-drop | Script de blocage via iptables |
| `<location>` | local | Exécuté sur l'agent victime |
| `<rules_id>` | 5763 | Règle force brute SSH (8 tentatives en 120s) |
| `<timeout>` | 180 | Durée du blocage en secondes (3 min) |

> **Règle 5763** : se déclenche après 8 tentatives SSH échouées en moins de 120 secondes — niveau de sévérité 10.

```bash
systemctl restart wazuh-manager
systemctl status wazuh-manager
```

### 6.3 Préparation de la machine victime

```bash
# Sur la machine victime (debian-wazu2)

# 1. Vérifier que SSH est actif
systemctl status ssh

# 2. Vérifier l'écoute sur le port 22
ss -tulpn | grep :22

# 3. Vérifier que l'authentification par mot de passe est activée
cat /etc/ssh/sshd_config | grep PasswordAuthentication
# → doit retourner : PasswordAuthentication yes

# Si nécessaire, l'activer :
micro /etc/ssh/sshd_config
# → PasswordAuthentication yes
systemctl restart sshd

# 4. Récupérer l'IP et le nom d'utilisateur ciblé
hostname -I
whoami
```

### 6.4 Préparation de la machine attaquante

```bash
# Sur la machine attaquante (debian-siem)

# Mise à jour
apt update && apt upgrade -y

# Installer Hydra (outil de force brute) et pwgen (générateur de mots de passe)
apt install -y hydra pwgen

# Générer une liste de 10 mots de passe aléatoires de 8 caractères
# ⚠️ Ne pas inclure le vrai mot de passe — on simule une attaque ÉCHOUÉE
pwgen 8 10 > password-list.txt
cat password-list.txt
```

### 6.5 Test de connectivité SSH (avant l'attaque)

```bash
# Depuis la machine attaquante — vérifier que SSH fonctionne vers la victime
ssh <utilisateur>@<IP-VICTIME>
# Accepter le fingerprint, entrer le mot de passe, puis se déconnecter
exit
```

### 6.6 Lancer l'attaque par force brute

```bash
# Depuis la machine attaquante
hydra -l <utilisateur> -P password-list.txt <IP-VICTIME> ssh
```

| Paramètre | Description |
|-----------|-------------|
| `-l <utilisateur>` | Login ciblé (ex: debian-wazu) |
| `-P password-list.txt` | Fichier de mots de passe |
| `<IP-VICTIME>` | IP de la machine cible |
| `ssh` | Protocole attaqué |

**Sortie attendue (attaque échouée) :**
```
[DATA] attacking ssh://<IP-VICTIME>:22/
[ERROR] all children were disabled due to too many connection errors
```

> Hydra peut afficher "too many connection errors" — c'est normal : l'Active Response a bloqué l'IP avant la fin des tentatives.

### 6.7 Analyser les résultats dans le Dashboard

Aller dans **Threat Hunting** → **Events** (filtrer par l'agent victime) :

| Règle | Description | Niveau |
|-------|-------------|--------|
| **5760** | Tentatives d'authentification SSH échouées | 5 |
| **5763** | Détection d'attaque par force brute SSH → déclenche l'Active Response | 10 |
| **651** | Blocage de l'IP par firewall-drop | — |
| **652** | Déblocage automatique de l'IP après expiration du timeout | — |

> Dans l'onglet **MITRE&ATT&CK**, l'attaque est classifiée **T1110** (Brute Force — Credential Access).

### 6.8 Vérifier le blocage et le déblocage automatique

**Depuis la machine attaquante — pendant le blocage actif :**

```bash
# Test SSH → doit être bloqué
ssh -o ConnectTimeout=5 <utilisateur>@<IP-VICTIME>
# → ssh: connect to host <IP-VICTIME> port 22: Connection timed out

# Test Ping → doit être bloqué (blocage total, pas seulement SSH)
ping -c 3 <IP-VICTIME>
# → 100% packet loss
```

**Depuis la machine victime — vérifier le blocage iptables :**

```bash
iptables -L -n -v | grep <IP-attaquante>
tail -f /var/ossec/logs/active-responses.log
```

**Après ~3 minutes — vérifier le déblocage automatique :**

```bash
# Depuis la machine attaquante
ping -c 3 <IP-VICTIME>
# → réponses normales (0% packet loss)

ssh <utilisateur>@<IP-VICTIME>
# → connexion SSH rétablie
```

**Résultats validés :**

| Résultat | Valeur |
|----------|--------|
| Règle 5763 déclenchée | Après 8 tentatives en ~120 secondes |
| Blocage effectif | En moins de 2 secondes |
| Trafic bloqué | 100% (SSH + ICMP) |
| Déblocage automatique | Après 181 secondes (~3 min) |

---

## 7. Intégration VirusTotal

L'intégration VirusTotal permet à Wazuh de soumettre automatiquement le hash SHA256 de tout fichier détecté par FIM à l'API VirusTotal. En cas de détection positive (fichier malveillant), une alerte est générée et une Active Response peut supprimer le fichier automatiquement.

### 7.1 Fonctionnement

```
Fichier détecté par FIM
        ↓
Hash SHA256 extrait
        ↓
API VirusTotal interrogée
        ↓
┌────────────────┬──────────────────────────┐
│  Non détecté   │  Détecté (malveillant)   │
│  → Aucune      │  → Alerte niveau 12      │
│    action      │  → Active Response       │
│                │    supprime le fichier   │
└────────────────┴──────────────────────────┘
```

| Composant | Rôle |
|-----------|------|
| **FIM (syscheck)** | Détecte les nouveaux fichiers et extrait leur hash |
| **Integration VirusTotal** | Envoie le hash à l'API et analyse la réponse |
| **Active Response** | Supprime automatiquement le fichier malveillant |

### 7.2 Prérequis

- Compte VirusTotal gratuit sur [virustotal.com](https://www.virustotal.com)
- FIM activé sur les agents (cf. section 5)
- `jq` installé sur les agents Linux
- Python 3 + PyInstaller sur les agents Windows

**Obtenir la clé API VirusTotal :**

1. Se connecter sur virustotal.com
2. Menu utilisateur → **Settings** → **API Key**
3. Copier la clé (limite : 4 requêtes/min avec le compte gratuit)

### 7.3 Configuration Manager — ossec.conf

```bash
micro /var/ossec/etc/ossec.conf
```

Ajouter le bloc `<integration>` dans `<ossec_config>` :

```xml
<integration>
  <name>virustotal</name>
  <api_key><VOTRE_CLE_API_VIRUSTOTAL></api_key>
  <group>syscheck</group>
  <alert_format>json</alert_format>
</integration>
```

> Le paramètre `<group>syscheck</group>` déclenche l'intégration sur toutes les alertes FIM. Pour cibler uniquement les nouveaux fichiers ajoutés, utiliser `<rule_id>554</rule_id>` à la place.

### 7.4 Règles personnalisées — local_rules.xml

```bash
micro /var/ossec/etc/rules/local_rules.xml
```

Ajouter avant la balise fermante `</group>` finale (ou créer un nouveau groupe) :

```xml
<group name="virustotal,">

  <rule id="100092" level="12">
    <if_sid>657</if_sid>
    <match>Successfully removed threat</match>
    <description>Active Response: fichier malveillant supprimé — $(parameters.alert.data.virustotal.source.file)</description>
  </rule>

  <rule id="100093" level="14">
    <if_sid>657</if_sid>
    <match>Error removing threat</match>
    <description>Active Response: échec de suppression — $(parameters.alert.data.virustotal.source.file)</description>
  </rule>

</group>
```

### 7.5 Active Response — Configuration Manager

Ajouter dans `/var/ossec/etc/ossec.conf` :

```xml
<command>
  <name>remove-threat</name>
  <executable>remove-threat.sh</executable>
  <timeout_allowed>no</timeout_allowed>
</command>

<active-response>
  <disabled>no</disabled>
  <command>remove-threat</command>
  <location>local</location>
  <rules_group>virustotal</rules_group>
</active-response>
```

Redémarrer le Manager :

```bash
systemctl restart wazuh-manager
```

### 7.6 Configuration Agent Linux

#### Installer jq

```bash
apt install -y jq
```

#### Activer FIM en temps réel sur le répertoire à surveiller

Dans `/var/ossec/etc/ossec.conf` de l'agent :

```xml
<syscheck>
  <disabled>no</disabled>
  <directories realtime="yes" check_all="yes">/root,/home,/tmp</directories>
</syscheck>
```

#### Créer le script de suppression Active Response

```bash
micro /var/ossec/active-response/bin/remove-threat.sh
```

```bash
#!/bin/bash

LOCAL=$(dirname "$0")
cd "$LOCAL" || exit
cd ../ || exit

read INPUT_JSON
FILENAME=$(echo "$INPUT_JSON" | jq -r '.parameters.alert.data.virustotal.source.file')
COMMAND=$(echo "$INPUT_JSON" | jq -r '.command')
LOG_FILE="$(pwd)/../logs/active-responses.log"

echo "$(date '+%Y/%m/%d %H:%M:%S') - remove-threat started" >> "${LOG_FILE}"
echo "$(date '+%Y/%m/%d %H:%M:%S') - Command: ${COMMAND}" >> "${LOG_FILE}"
echo "$(date '+%Y/%m/%d %H:%M:%S') - File: ${FILENAME}" >> "${LOG_FILE}"

if [ "${COMMAND}" = "add" ]; then
    if rm -f "${FILENAME}"; then
        echo "$(date '+%Y/%m/%d %H:%M:%S') Successfully removed threat: ${FILENAME}" >> "${LOG_FILE}"
    else
        echo "$(date '+%Y/%m/%d %H:%M:%S') Error removing threat: ${FILENAME}" >> "${LOG_FILE}"
    fi
fi
```

Appliquer les permissions :

```bash
chmod 750 /var/ossec/active-response/bin/remove-threat.sh
chown root:wazuh /var/ossec/active-response/bin/remove-threat.sh
```

Redémarrer l'agent :

```bash
systemctl restart wazuh-agent
```

### 7.7 Configuration Agent Windows

#### Activer FIM en temps réel

Dans `C:\Program Files (x86)\ossec-agent\ossec.conf` :

```xml
<syscheck>
  <disabled>no</disabled>
  <directories realtime="yes">C:\Users\<NOM_UTILISATEUR>\Downloads,C:\Users\<NOM_UTILISATEUR>\Desktop</directories>
</syscheck>
```

#### Créer le script de suppression Active Response

Installer Python 3 avec l'option **"Add Python to PATH"** activée, puis :

```powershell
pip install pyinstaller
```

Créer `C:\remove-threat.py` :

```python
#!/usr/bin/python3
import sys
import json
import os
import datetime

LOG_FILE = "C:\\Program Files (x86)\\ossec-agent\\active-response\\active-responses.log"

def write_log(msg):
    with open(LOG_FILE, "a") as f:
        f.write(f"{datetime.datetime.now()} - {msg}\n")

if __name__ == "__main__":
    write_log("remove-threat started")
    input_str = sys.stdin.readline()
    try:
        data = json.loads(input_str)
        command = data.get("command", "")
        filename = data["parameters"]["alert"]["data"]["virustotal"]["source"]["file"]
        write_log(f"Command: {command} | File: {filename}")
        if command == "add":
            if os.path.exists(filename):
                os.remove(filename)
                write_log(f"Successfully removed threat: {filename}")
            else:
                write_log(f"File not found: {filename}")
    except Exception as e:
        write_log(f"Error removing threat: {e}")
```

Compiler en exécutable :

```powershell
pyinstaller -F C:\remove-threat.py
```

Déplacer l'exécutable :

```powershell
Move-Item -Path C:\dist\remove-threat.exe `
  -Destination "C:\Program Files (x86)\ossec-agent\active-response\bin\remove-threat.exe"
```

Redémarrer l'agent :

```powershell
Restart-Service -Name wazuh
```

### 7.8 Test de détection — Fichier EICAR

Le fichier EICAR est un fichier de test standard reconnu comme malveillant par tous les antivirus et VirusTotal, sans danger réel.

#### Test sur Linux (agent)

```bash
# Télécharger le fichier EICAR dans un répertoire surveillé par FIM
curl -Lo /root/eicar.com https://secure.eicar.org/eicar.com
sleep 10
# Vérifier si le fichier a été supprimé par Active Response
ls -la /root/eicar.com
# → No such file or directory (suppression réussie)
```

#### Test sur Windows (agent)

```powershell
# Télécharger le fichier EICAR dans le répertoire surveillé
Invoke-WebRequest -Uri https://secure.eicar.org/eicar.com.txt `
  -OutFile "C:\Users\<NOM_UTILISATEUR>\Downloads\eicar.txt"
Start-Sleep -Seconds 10
# Vérifier si le fichier a été supprimé
Test-Path "C:\Users\<NOM_UTILISATEUR>\Downloads\eicar.txt"
# → False (suppression réussie)
```

### 7.9 Vérification Dashboard

1. Aller dans **Threat Hunting** → **Events**
2. Filtrer par `rule.groups: virustotal`

**Alertes attendues :**

| Règle | Niveau | Description |
|-------|--------|-------------|
| `87105` | 12 | VirusTotal: fichier positif — hash détecté comme malveillant |
| `87103` | 3 | VirusTotal: fichier non détecté (hash inconnu) |
| `100092` | 12 | Active Response: fichier malveillant supprimé avec succès |
| `100093` | 14 | Active Response: échec de suppression |

**Champs clés dans l'alerte VirusTotal :**

| Champ | Description |
|-------|-------------|
| `data.virustotal.source.file` | Chemin complet du fichier détecté |
| `data.virustotal.malicious` | Nombre de moteurs ayant détecté le fichier |
| `data.virustotal.total` | Nombre total de moteurs ayant analysé le fichier |
| `data.virustotal.permalink` | Lien direct vers le rapport VirusTotal |
| `data.virustotal.sha256` | Hash SHA256 soumis |

**Vérifier les logs Active Response sur l'agent :**

```bash
# Linux
tail -f /var/ossec/logs/active-responses.log

# Windows (PowerShell)
Get-Content "C:\Program Files (x86)\ossec-agent\active-response\active-responses.log" -Wait
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
