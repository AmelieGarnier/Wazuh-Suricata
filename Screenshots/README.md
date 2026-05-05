# Screenshots

Placer ici les captures d'écran référencées dans les runbooks.

## Wazuh (13 captures)

| Fichier | Section | Description |
|---------|---------|-------------|
| `01-dashboard-home.png` | Wazuh §3.4 | Page d'accueil du Dashboard après connexion |
| `02-agents-list.png` | Wazuh §4.3 | Liste des agents connectés (Linux + Windows) |
| `15-agent-connected-dashboard.png` | Wazuh §4.3 | Agent 001 — statut Active, version, OS |
| `03-fim-events-list.png` | Wazuh §5 | Liste des événements FIM |
| `04-fim-event-detail-windows.png` | Wazuh §5.1 | Détail d'un événement FIM Windows |
| `05-fim-whodata-detail.png` | Wazuh §5.3 | Champs Whodata (utilisateur + processus) |
| `06-threat-hunting.png` | Wazuh §6.3 | Threat Hunting — 80 événements Authentication failure |
| `07-brute-force-alert-5763.png` | Wazuh §6.2 | Alerte règle 5763 brute-force SSH |
| `08-active-response-block.png` | Wazuh §6.3 | Confirmation blocage Active Response |
| `09-mitre-attack-t1110.png` | Wazuh §6 | MITRE ATT&CK T1110 dans le Dashboard |
| `10-virustotal-alert-positive.png` | Wazuh §7.2 | Alerte VirusTotal — fichier détecté (66/71 moteurs) |
| `11-virustotal-detail-hash.png` | Wazuh §7.2 | Détail VT : permalink + positives/total |
| `14-manager-services-status.png` | Wazuh §3.4 | systemctl status des 4 services |

## Suricata (12 captures)

| Fichier | Section | Description |
|---------|---------|-------------|
| `suricata-01-architecture.png` | Suricata §2 | Architecture Suricata — vue d'ensemble |
| `suricata-02-afpacket-config.png` | Suricata §3.2 | Configuration AF_PACKET dans suricata.yaml |
| `suricata-03-list-sources.png` | Suricata §4.2 | Sources de règles disponibles (list-sources) |
| `18-suricata-rules-loaded.png` | Suricata §4.3 | Confirmation 46 334 règles chargées |
| `suricata-04-localfile-config.png` | Suricata §5.1 | Bloc localfile dans ossec.conf |
| `suricata-05-eve-permissions.png` | Suricata §5.3 | Permissions du fichier eve.json |
| `16-suricata-service-status.png` | Suricata §6.2 | systemctl status suricata |
| `suricata-06-test-curl.png` | Suricata §7.1 | Test curl testmyids.com — réponse root |
| `17-eve-json-alert-jq.png` | Suricata §7.2 | Sortie eve.json filtrée avec jq |
| `19-suricata-gpl-alert-wazuh.png` | Suricata §7.3 | Alerte GPL ATTACK_RESPONSE dans Wazuh |
| `20-suricata-threat-hunting-filter.png` | Suricata §7.3 | Threat Hunting filtré event_type:alert |
| `13-suricata-alerts-wazuh.png` | Suricata §7.3 | Alertes Suricata dans le Dashboard Wazuh |
