---
description: Applique toutes les corrections d'audit sur les runbooks Wazuh et Suricata, puis commit et push
allowed-tools: Read, Write, Bash(git add:*), Bash(git status:*), Bash(git commit:*), Bash(git push:*), Bash(git diff:*)
---

Applique les corrections suivantes sur les fichiers `docs/wazuh-runbook.md` et `docs/suricata-runbook.md`, puis commit et push les changements.

---

## CORRECTIONS — docs/wazuh-runbook.md

### 1. Ajouter vm.max_map_count dans la section Prérequis

Ajouter un bloc **avant** la section "### Outils système" :

```markdown
### Paramètre noyau requis (OpenSearch)

OpenSearch (Wazuh Indexer) requiert un `vm.max_map_count` élevé. Sans ce paramètre, le service refuse de démarrer.

```bash
sysctl -w vm.max_map_count=262144
echo "vm.max_map_count=262144" >> /etc/sysctl.conf
```
```

### 2. Corriger le port manquant dans les prérequis

Remplacer :
```
Ports disponibles : `443` (Dashboard), `9200` (Indexer), `1514-1515` (Manager)
```
Par :
```
Ports disponibles : `443` (Dashboard), `9200` (Indexer), `1514-1515` (Manager), `55000` (API Wazuh Manager)
```

### 3. Corriger les URLs du certs-tool de 4.8 → 4.13

Remplacer :
```bash
curl -sO https://packages.wazuh.com/4.8/wazuh-certs-tool.sh
curl -sO https://packages.wazuh.com/4.8/config.yml
```
Par :
```bash
curl -sO https://packages.wazuh.com/4.13/wazuh-certs-tool.sh
curl -sO https://packages.wazuh.com/4.13/config.yml
```

Et mettre à jour le commentaire associé :
```bash
# Télécharger les outils — version 4.13
```

### 4. Ajouter la désactivation du dépôt après l'installation des paquets

Ajouter après la section `apt install -y wazuh-indexer wazuh-manager wazuh-dashboard` :

```markdown
### Désactiver le dépôt après installation

Pour éviter une mise à jour automatique non maîtrisée :

```bash
sed -i "s/^deb/#deb/" /etc/apt/sources.list.d/wazuh.list && apt-get update
```

> Pour réactiver lors d'une mise à jour intentionnelle : `sed -i "s/^#deb/deb/" /etc/apt/sources.list.d/wazuh.list && apt-get update`
```

### 5. Corriger la version du module Filebeat (0.4 → 0.5)

Remplacer :
```bash
curl -s https://packages.wazuh.com/4.x/filebeat/wazuh-filebeat-0.4.tar.gz \
  | tar -xvz -C /usr/share/filebeat/module
```
Par :
```bash
# Installer le module Wazuh — version 0.5
curl -s https://packages.wazuh.com/4.x/filebeat/wazuh-filebeat-0.5.tar.gz \
  | tar -xvz -C /usr/share/filebeat/module
```

### 6. Corriger l'URL du filebeat.yml de 4.8 → 4.13

Remplacer :
```bash
curl -so /etc/filebeat/filebeat.yml \
  https://packages.wazuh.com/4.8/tpl/wazuh/filebeat/filebeat.yml
```
Par :
```bash
# Télécharger la configuration — version 4.13
curl -so /etc/filebeat/filebeat.yml \
  https://packages.wazuh.com/4.13/tpl/wazuh/filebeat/filebeat.yml
```

### 7. Supprimer l'URL du wazuh-template.json (obsolète en 4.13)

Supprimer ces deux lignes :
```bash
curl -so /etc/filebeat/wazuh-template.json \
  https://raw.githubusercontent.com/wazuh/wazuh/v4.8.2/extensions/elasticsearch/7.x/wazuh-template.json
chmod go+r /etc/filebeat/wazuh-template.json
```

> En Wazuh 4.9+, le template est inclus dans le module wazuh-filebeat-0.5 — le téléchargement séparé n'est plus nécessaire.

### 8. Ajouter la section 3.5 — configuration wazuh.yml (API Manager)

Ajouter après la section 3.4 Wazuh Dashboard, une nouvelle section :

```markdown
### 3.5 Configurer la connexion Dashboard → Manager (API)

```bash
micro /usr/share/wazuh-dashboard/data/wazuh/config/wazuh.yml
```

```yaml
hosts:
  - default:
      url: https://192.168.111.62
      port: 55000
      username: wazuh-wui
      password: wazuh-wui
      run_as: true
```

```bash
systemctl restart wazuh-dashboard
```
```

### 9. Ajouter un avertissement changement de mot de passe

Sous la ligne `**Accès web :** ...` ajouter :

```
> ⚠️ **Changer le mot de passe admin** après la première connexion via le Dashboard → Administration → Security → Users.
```

### 10. Compléter le bloc <indexer> avec les certificats SSL (section 5.4)

Remplacer le bloc XML incomplet :
```xml
<indexer>
  <enabled>yes</enabled>
  <hosts>
    <host>https://192.168.111.62:9200</host>
  </hosts>
  ...
</indexer>
```
Par :
```xml
<indexer>
  <enabled>yes</enabled>
  <hosts>
    <host>https://192.168.111.62:9200</host>
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

### 11. Ajouter vm.max_map_count dans le troubleshooting Indexer

Dans la section troubleshooting `Connection refused port 9200`, ajouter après l'étape 2 :

```bash
# 3. Vérifier vm.max_map_count (cause la plus fréquente)
sysctl vm.max_map_count
# → doit retourner 262144. Si inférieur :
sysctl -w vm.max_map_count=262144
```
Et renuméroter les étapes suivantes (4, 5, 6).

---

## CORRECTIONS — docs/suricata-runbook.md

### 1. Supprimer les packages inutiles de l'installation

Remplacer le bloc d'installation :
```bash
# Suricata + jq (traitement JSON)
apt install -y suricata jq

# Gestionnaire de dépôts
apt install -y software-properties-common

# Python (outils d'intégration)
apt install -y python3-pip
```
Par :
```bash
# Suricata + jq (traitement JSON)
apt install -y suricata jq
```

Et supprimer la table de versions qui référence software-properties-common et python3-pip — la remplacer par :

| Paquet | Version |
|--------|---------|
| suricata | 6.0.10 |
| jq | 1.6-2.1 |

### 2. Ajouter la désactivation GRO/LRO après l'installation

Ajouter après l'installation des paquets, une nouvelle section :

```markdown
### Désactiver les offloads réseau (performance AF_PACKET)

La désactivation de GRO/LRO évite des problèmes de checksum et garantit une capture complète des paquets :

```bash
ethtool -K ens192 gro off lro off
```

Rendre persistant via systemd :

```bash
cat > /etc/systemd/system/disable-offload.service << 'EOF'
[Unit]
Description=Disable GRO/LRO on ens192
After=network.target

[Service]
Type=oneshot
ExecStart=/sbin/ethtool -K ens192 gro off lro off
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF

systemctl enable disable-offload
systemctl start disable-offload
```
```

### 3. Ajouter la section default-rule-path dans la configuration

Ajouter une nouvelle section **3.5** (avant la section EVE-JSON qui devient 3.6) :

```markdown
### 3.5 Chemin des règles

Vérifier que cette section est présente dans `suricata.yaml` :

```yaml
default-rule-path: /var/lib/suricata/rules

rule-files:
  - suricata.rules
```

> Sans cette configuration, `suricata-update` dépose bien les règles mais Suricata démarre avec **0 règle chargée**.
```

### 4. Corriger la table des fichiers (section 4.5)

Remplacer la description de `/etc/suricata/rules/` :

Remplacer :
```
| Règles distribuées | `/etc/suricata/rules/` |
```
Par :
```
| Règles locales custom (non gérées par suricata-update) | `/etc/suricata/rules/` |
```

### 5. Corriger le chown dans la section 5.3

Remplacer :
```bash
chmod 644 /var/log/suricata/eve.json
chown root:root /var/log/suricata/eve.json
systemctl restart wazuh-manager
```
Par :
```bash
chmod 644 /var/log/suricata/eve.json
# Ne pas changer l'owner — Suricata doit pouvoir écrire dans ce fichier
# Vérifier l'owner actuel : ls -la /var/log/suricata/eve.json
systemctl restart wazuh-manager
```

### 6. Corriger le même chown dans le troubleshooting (section 9)

Dans la section "Wazuh ne reçoit pas les alertes Suricata", remplacer :
```bash
chmod 644 /var/log/suricata/eve.json
chown root:root /var/log/suricata/eve.json
systemctl restart wazuh-manager
```
Par :
```bash
chmod 644 /var/log/suricata/eve.json
# Ne pas modifier l'owner : Suricata doit conserver l'accès en écriture
systemctl restart wazuh-manager
```

---

## COMMIT

Une fois toutes les corrections appliquées, exécuter :

```bash
git add docs/wazuh-runbook.md docs/suricata-runbook.md
git status
git commit -m "fix(runbooks): corrections audit — 8 bugs wazuh + 6 bugs suricata

Wazuh:
- URLs certs-tool et filebeat.yml 4.8 → 4.13
- Module wazuh-filebeat 0.4 → 0.5
- Supprimer wazuh-template.json obsolète
- Ajouter vm.max_map_count=262144 (prérequis + troubleshooting)
- Ajouter port 55000 dans prérequis
- Compléter bloc <indexer> avec certs SSL
- Ajouter section wazuh.yml API config (3.5)
- Ajouter désactivation repo post-install

Suricata:
- Supprimer packages inutiles (software-properties-common, python3-pip)
- Ajouter désactivation GRO/LRO (ethtool + service systemd)
- Ajouter section default-rule-path dans suricata.yaml
- Corriger description /etc/suricata/rules/
- Corriger chown root:root → garder owner suricata (x2)"
git push
```

