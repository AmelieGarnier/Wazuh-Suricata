# Runbook Wazuh — SIEM / XDR

> **Environnement :** Debian 13 — IP serveur : `<IP-SERVEUR>`
> **Version Wazuh :** 4.13.x

---

## Présentation de Wazuh

Wazuh est la plateforme de cybersécurité open-source la plus adoptée au monde. Elle unifie dans une seule solution deux paradigmes distincts : le **SIEM** (Security Information and Event Management) et le **XDR** (Extended Detection and Response). Elle excelle dans les domaines suivants :

- **Détection des intrusions (IDS)** — Analyse comportementale et détection d'anomalies
- **Surveillance de l'intégrité des fichiers (FIM)** — Détection de modifications non autorisées
- **Réponse aux incidents** — Automatisation des actions de remédiation (Active Response)
- **Conformité réglementaire** — PCI-DSS, GDPR, HIPAA, CIS, NIS2
- **Analyse de vulnérabilités** — Corrélation CVE via Wazuh CTI
- **Détection de malwares** — Intégration VirusTotal et MISP
- **Threat Hunting** — Mapping MITRE ATT&CK natif

### Architecture all-in-one (single-node)

Tous les composants sont installés sur un seul serveur. Adapté aux environnements de test, petites infrastructures (< 100 agents) et POC.

| Composant | Rôle |
|-----------|------|
| **Wazuh Indexer** (OpenSearch) | Stockage, indexation et recherche des données |
| **Wazuh Manager** | Cœur du système — analyse, corrélation, alertes |
| **Filebeat** | Collecte et transmission des logs vers l'Indexer |
| **Wazuh Dashboard** | Interface web de visualisation et gestion |

> Pour les environnements de production à grande échelle (> 500 agents), une architecture **multi-node** avec serveurs dédiés est recommandée pour garantir HA et performances.

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
8. [Intégration Suricata](#8-intégration-suricata)
9. [Opérations courantes](#9-opérations-courantes)
10. [Troubleshooting](#10-troubleshooting)

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

Exemple de sortie attendue :
```
inet 192.168.1.50/24 brd 192.168.1.255 scope global enp0s3
```

> Noter cette IP — elle remplace `<IP-SERVEUR>` dans toutes les commandes suivantes.

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

> Si `micro` échoue à cause d'un problème DNS temporaire, corriger le DNS avant de réessayer (voir section Troubleshooting).

### Corriger le DNS si nécessaire

```bash
echo "nameserver 8.8.8.8" >> /etc/resolv.conf
apt update
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
# Télécharger les outils — version 4.13
curl -sO https://packages.wazuh.com/4.13/wazuh-certs-tool.sh
curl -sO https://packages.wazuh.com/4.13/config.yml

# Vérifier la présence des fichiers
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

> Les guillemets autour de l'IP sont obligatoires. Les noms `node-1`, `wazuh-1` et `dashboard` doivent être conservés tels quels.

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
```

> Alternative pour écouter sur toutes les interfaces : `network.host: "0.0.0.0"`

```bash
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
# Ce script configure les rôles, permissions, utilisateurs internes et politiques SSL/TLS
/usr/share/wazuh-indexer/bin/indexer-security-init.sh
```

> **Si `runuser : commande introuvable`** lors de `indexer-security-init.sh` : vérifier que `export PATH=$PATH:/sbin:/usr/sbin` a bien été exécuté.

```bash
# Tester — résultat attendu : informations JSON sur le cluster
curl -k -u admin:admin https://<IP-SERVEUR>:9200

# Lister les nœuds du cluster
curl -k -u admin:admin https://<IP-SERVEUR>:9200/_cat/nodes?v
```

Résultats attendus :
- Première commande → objet JSON avec `name`, `cluster_name`, `version`
- Deuxième commande → tableau tabulaire des nœuds actifs avec statistiques

### 3.2 Wazuh Manager

```bash
systemctl daemon-reload
systemctl enable wazuh-manager
systemctl start wazuh-manager
systemctl status wazuh-manager
```

Résultat attendu : `Active: active (running)` en vert.

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
```

> ⚠️ **Note de sécurité** : en production, remplacer `admin/admin` par des identifiants personnalisés.

```bash
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
```

Résultat attendu : `Index setup finished.`
L'erreur Kibana/port 5601 est normale — ignorer.

```bash
# Démarrer
systemctl daemon-reload
systemctl enable filebeat
systemctl start filebeat

# Tester — résultat attendu : "talk to server... OK"
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

# Vérifier les 3 services — tous doivent afficher "active (running)"
systemctl status wazuh-manager wazuh-indexer wazuh-dashboard
```

**Accès web :** `https://<IP-SERVEUR>` — identifiants par défaut : `admin / admin`

> ⚠️ **Changer le mot de passe admin obligatoirement** :
> Menu hamburger (☰) → **Security** → **Internal users** → sélectionner `admin` → **Edit** → changer le mot de passe → **Save**

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

Depuis l'interface web Wazuh → **Agents** → **Deploy new agent** → sélectionner **DEB amd64** → renseigner l'IP du Manager et le nom de l'agent → copier la commande générée.

```bash
# Sur la machine agent
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

Résultat attendu : `Active: active (running)` en vert. L'agent apparaît dans le Dashboard avec le statut **Active** (indicateur vert).

### 4.2 Agent Windows 10 (PowerShell en Admin)

Depuis l'interface web Wazuh → **Agents** → **Deploy new agent** → sélectionner **Windows** → renseigner l'IP du Manager et le nom → copier la commande générée.

```powershell
Invoke-WebRequest -Uri https://packages.wazuh.com/4.x/windows/wazuh-agent-4.13.1-1.msi `
  -OutFile $env:tmp\wazuh-agent.msi

msiexec.exe /i $env:tmp\wazuh-agent.msi /q `
  WAZUH_MANAGER='<IP-SERVEUR>' `
  WAZUH_AGENT_GROUP='default' `
  WAZUH_AGENT_NAME='<nom-agent>'

NET START Wazuh
```

Résultat attendu : message `Le service WazuhSvc a démarré.`

### 4.3 Vérifier les agents depuis le Manager

```bash
# Lister les agents — doit afficher tous les agents avec statut Active
/var/ossec/bin/agent_control -l

# Détails d'un agent
/var/ossec/bin/agent_control -i 001
```

---

## 5. Configuration FIM

Le **File Integrity Monitoring (FIM)** surveille les modifications de fichiers et répertoires en temps réel. Les alertes FIM sont générées lors de créations, modifications, suppressions, et changements de permissions.

| Mode | Temps réel | Identité auteur | Impact performance | Usage recommandé |
|------|-----------|-----------------|-------------------|-----------------|
| Basic (scheduled) | Non | Non | Faible | Surveillance générale |
| Realtime | Oui | Non | Moyen | Fichiers critiques |
| Whodata | Oui | Oui (utilisateur, processus) | Élevé | Audit de sécurité avancé |

### 5.1 FIM sur Windows

Fichier de configuration : `C:\Program Files (x86)\ossec-agent\ossec.conf`

Ouvrir en tant qu'administrateur avec le Bloc-notes.

```xml
<syscheck>
  <directories check_all="yes" realtime="yes" report_changes="yes">
    C:\Users\<utilisateur>\Documents
  </directories>
</syscheck>
```

Paramètres disponibles :
- `realtime="yes"` → détection immédiate des modifications
- `report_changes="yes"` → signale toutes les modifications
- `check_all="yes"` → surveille toutes les métadonnées (permissions, propriétaire, taille…)
- `check_md5="yes"` / `check_sha1="yes"` / `check_sha256="yes"` → empreintes cryptographiques

Redémarrer l'agent via `Win+R` → `services.msc` → service **Wazuh** → Redémarrer.

### 5.2 FIM sur Linux

Fichier de configuration : `/var/ossec/etc/ossec.conf`

```xml
<syscheck>
  <directories check_all="yes" realtime="yes" report_changes="yes">
    /etc,/usr/bin,/usr/sbin
  </directories>
</syscheck>
```

Répertoires critiques recommandés : `/etc`, `/bin`, `/sbin`, `/usr/bin`, `/home`, `/var/www`

```bash
systemctl restart wazuh-agent
systemctl status wazuh-agent
```

Vérifier dans le Dashboard → **Threat Hunting** → **Events** : les événements FIM apparaissent avec :
- Chemin complet du fichier
- Type d'opération (création, modification, suppression)
- Date et heure
- Checksums (MD5, SHA1, SHA256)

### 5.3 Mode Whodata (audit avancé)

Le mode Whodata enregistre quel utilisateur et quel processus ont modifié un fichier.

**Windows :**

```xml
<syscheck>
  <directories check_all="yes" whodata="yes">
    C:\Users\<utilisateur>\Documents
  </directories>
</syscheck>
```

Informations supplémentaires disponibles dans le Dashboard :
- Nom d'utilisateur Windows (SID et nom)
- Nom du processus ayant effectué la modification
- ID du processus (PID)
- Chemin complet de l'exécutable
- Type d'accès (lecture, écriture, suppression)

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
  <directories whodata="yes">/root</directories>
  <directories whodata="yes">/home/<utilisateur></directories>
</syscheck>
```

```bash
systemctl restart wazuh-agent
```

Informations disponibles sous Linux : UID, PID, PPID, GID, nom du processus, arguments de la ligne de commande, contexte SELinux.

### 5.4 Résolution du problème de connexion à l'Indexer

Si le Dashboard affiche une erreur de connexion à l'Indexer (cause : `0.0.0.0` au lieu de l'IP réelle dans la config du Manager) :

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
Machine attaquante → SSH → Machine victime (agent)
  → Wazuh Manager (règle 5763 déclenchée après 8 tentatives)
    → firewall-drop → Blocage IP via iptables
      → Déblocage automatique après timeout
```

### 6.2 Flux d'information

1. L'agent Wazuh sur la victime envoie les logs SSH au Manager (port 1514)
2. Le Manager corrèle les événements et détecte le pattern d'attaque
3. La règle **5763** (brute-force SSH, sévérité 10) se déclenche
4. Une commande `firewall-drop` est envoyée à l'agent victime
5. L'agent bloque l'IP via iptables — **tout le trafic est bloqué**, pas seulement SSH
6. Après le timeout, l'IP est automatiquement débloquée (règle 652)

### 6.3 Configuration sur le Manager

```bash
micro /var/ossec/etc/ossec.conf
```

Vérifier d'abord que la commande `firewall-drop` est bien définie dans `<ossec_config>` :

```xml
<command>
  <name>firewall-drop</name>
  <executable>firewall-drop</executable>
  <timeout_allowed>yes</timeout_allowed>
</command>
```

Ajouter ensuite la réponse active :

```xml
<active-response>
  <command>firewall-drop</command>
  <location>local</location>
  <rules_id>5763</rules_id>
  <timeout>600</timeout>
</active-response>
```

| Paramètre | Valeur | Description |
|-----------|--------|-------------|
| `<command>` | firewall-drop | Commande à exécuter |
| `<location>` | local | S'exécute sur l'agent victime lui-même |
| `<rules_id>` | 5763 | Règle de détection brute-force SSH |
| `<timeout>` | 600 | Durée du blocage en secondes (10 min) |

> **Règle 5763** : se déclenche après 8 tentatives échouées en 120 secondes.

```bash
systemctl restart wazuh-manager
systemctl status wazuh-manager
```

> Si le service ne redémarre pas : vérifier la syntaxe XML avec `tail -f /var/ossec/logs/ossec.log`

### 6.4 Alertes attendues dans le Dashboard

| Règle | Description |
|-------|-------------|
| 5760 | Tentatives d'authentification SSH échouées |
| 5763 | Détection d'attaque par force brute SSH |
| 651 | Blocage de l'IP par firewall-drop |
| 652 | Déblocage de l'IP après expiration du timeout |

Les alertes sont mappées automatiquement à la technique **MITRE ATT&CK T1110** (Brute Force — Credential Access), visible dans l'onglet **MITRE ATT&CK** du Dashboard.

### 6.5 Vérifier le blocage

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

Résultats attendus pendant le blocage :
- SSH → `Connection timed out`
- Ping → 100% packet loss (tout le trafic bloqué)

---

## 7. Intégration VirusTotal

Wazuh envoie automatiquement les hash MD5/SHA256 des fichiers détectés par FIM à l'API VirusTotal pour identifier les malwares via l'intelligence collective de plus de 70 moteurs antivirus.

**Bénéfices :**
- Validation externe des fichiers suspects détectés par FIM
- Réduction des faux positifs grâce à la réputation des fichiers
- Détection de malwares zero-day via signatures communautaires
- Contexte enrichi pour l'investigation d'incidents

### 7.1 Récupérer la clé API VirusTotal

1. Se connecter sur [https://www.virustotal.com](https://www.virustotal.com)
2. Cliquer sur l'**icône de profil** en haut à droite
3. Ouvrir **API Keys**
4. Copier la clé affichée

> ⚠️ **Sécurité — ne jamais :**
> - Coller la clé dans le terminal sans la masquer
> - La committer dans un dépôt Git (même privé)
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
  <name>virustotal</name>
  <api_key>VOTRE_CLE_API_ICI</api_key>
  <group>syscheck</group>
  <alert_format>json</alert_format>
</integration>
```

> ⚠️ Remplacer `VOTRE_CLE_API_ICI` par la valeur copiée depuis VirusTotal. Ne pas entourer la clé de chevrons `< >`.

```bash
systemctl restart wazuh-manager
```

### 7.3 Vérifier l'intégration

```bash
# Vérifier les logs d'intégration
tail -f /var/ossec/logs/integrations.log

# Tester — déposer un fichier dans un répertoire surveillé par FIM
touch /etc/test-fim-virustotal
```

Les alertes VirusTotal apparaissent dans le Dashboard sous **Threat Hunting** avec le groupe `virustotal`, enrichies du score de détection par les différents moteurs.

---

## 8. Intégration Suricata

Suricata est un IDS/IPS réseau open-source qui capture et analyse le trafic en temps réel. Associé à Wazuh, il offre une vision à 360° de la sécurité : événements endpoints (Wazuh) + événements réseau (Suricata).

### 8.1 Architecture de corrélation

```
Trafic réseau → Suricata (IDS) → eve.json
  → Wazuh logcollector → Wazuh Manager (analyse)
    → Wazuh Dashboard (Threat Hunting → Events)
```

| Événement Suricata | Corrélation Wazuh | Action automatique |
|-------------------|-------------------|-------------------|
| Scan de ports détecté | Activité réseau anormale | Active Response bloque l'IP |
| Alerte exploitation CVE | FIM détecte modification fichier | Alerte haute sévérité |
| Trafic vers IP malveillante | Logs firewall + processus | Blocage IP + analyse |

### 8.2 Configurer Wazuh pour lire les logs Suricata

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

### 8.3 Corriger les permissions et redémarrer

```bash
chmod 644 /var/log/suricata/eve.json
# Ne pas changer l'owner — Suricata doit pouvoir écrire dans ce fichier
systemctl restart wazuh-manager
```

### 8.4 Résoudre les erreurs de décodage JSON

Si des erreurs `Too many fields for JSON decoder` apparaissent dans `/var/ossec/logs/ossec.log` (causées par les événements `stats` volumineux) :

```bash
micro /var/ossec/etc/internal_options.conf
# Modifier ou ajouter :
# analysisd.decoder_order_size=512

systemctl restart wazuh-manager
```

> Ces erreurs sur les événements `stats` sont normales et n'empêchent pas la transmission des alertes de sécurité.

### 8.5 Vérifier l'intégration

```bash
# Vérifier que Wazuh lit le fichier — utiliser -a si grep signale "fichiers binaires"
grep -ia "eve.json" /var/ossec/logs/ossec.log

# Vérifier les alertes en temps réel
tail -f /var/log/suricata/eve.json | jq 'select(.event_type == "alert")'
```

Pour consulter les alertes Suricata dans le Dashboard : **Threat Hunting** → **Events** → filtrer par `data.event_type: alert`.

---

## 9. Opérations courantes

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

## 10. Troubleshooting

### `sysctl : commande introuvable` ou `runuser : commande introuvable`

```bash
export PATH=$PATH:/sbin:/usr/sbin
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

### Erreur DNS temporaire

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

# 2. Vérifier via journalctl
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

```bash
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
# Sur l'AGENT
cat /var/ossec/etc/ossec.conf | grep -A 5 "<client>"
telnet <IP-SERVEUR> 1514
tail -f /var/ossec/logs/ossec.log
systemctl restart wazuh-agent

# Sur le MANAGER
ss -tulpn | grep -E '1514|1515'
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
systemctl status wazuh-dashboard
ss -tulpn | grep :443
cat /etc/wazuh-dashboard/opensearch_dashboards.yml | grep server.host
tail -50 /var/log/wazuh-dashboard/wazuh-dashboard.log
systemctl restart wazuh-dashboard
```

---

### Active Response ne bloque pas l'IP

```bash
# Sur la machine VICTIME (agent)
iptables -L -n -v | grep <IP-attaquante>
tail -f /var/ossec/logs/active-responses.log
/var/ossec/active-response/bin/firewall-drop add - <IP-attaquante>
iptables -L -n | grep <IP-attaquante>
/var/ossec/active-response/bin/firewall-drop delete - <IP-attaquante>
```

---

### `grep` retourne "fichiers binaires correspondent"

Utiliser l'option `-a` pour forcer le traitement comme texte :

```bash
grep -ia "terme_recherché" /var/ossec/logs/ossec.log
```

---

## Références

| Ressource | URL |
|-----------|-----|
| Site officiel Wazuh | https://wazuh.com |
| Documentation | https://documentation.wazuh.com |
| GitHub | https://github.com/wazuh |
| Communauté Discord | https://discord.gg/rg9eZTtC7W |
| Guide d'installation rapide | https://documentation.wazuh.com/current/quickstart.html |
