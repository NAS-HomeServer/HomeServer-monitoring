# Monitoring HomeServer

Stack de monitoring complète pour NAS Synology, déployée via Docker Compose et Ansible, avec CI/CD GitHub Actions.

## Stack

| Service | Version | Port | Rôle |
|---|---|---|---|
| Prometheus | v3.1.0 | 9090 | Base de données de métriques |
| Grafana | v11.6.1 | 3001 | Visualisation et dashboards |
| Node Exporter | v1.9.0 | 9100 | Métriques OS (CPU, RAM, disques, réseau) |
| cAdvisor | v0.49.1 | 8080 | Métriques des conteneurs Docker |
| SNMP Exporter | — | 9116 | Métriques matérielles Synology (optionnel) |

## Prérequis

- NAS Synology avec Docker et Docker Compose installés
- Python 3 + Ansible (pour le déploiement IaC)
- Runner GitHub Actions auto-hébergé sur le NAS (pour la CI/CD)
- Répertoire `/volume1/docker` existant

## Déploiement

### Via Ansible (recommandé)

```bash
ansible-playbook -i ansible/inventory/hosts.ini ansible/playbooks/deploy-monitoring.yml
```

Le playbook :
1. Crée les répertoires de données nécessaires
2. Purge les conteneurs orphelins
3. Déploie la stack via `docker-compose`

### Via Docker Compose

```bash
docker-compose up -d
```

### Via GitHub Actions

Le workflow se déclenche automatiquement sur chaque push sur `main`, ou manuellement depuis l'onglet **Actions**.

Il :
1. Télécharge le code via l'API GitHub
2. Build une image Docker embarquant Ansible
3. Exécute le playbook depuis ce conteneur en montant le socket Docker

## Configuration

### Prometheus — `config/prometheus.yml`

Intervalles globaux : `15s` scrape / `15s` évaluation.

Jobs configurés :
- `synology-nas` — node-exporter (`node-exporter:9100`)
- `docker-containers` — cAdvisor (`cadvisor:8080`)
- `prometheus` — auto-monitoring (`localhost:9090`)
- `synology-snmp` — SNMP Exporter *(commenté par défaut, voir section SNMP)*

### SNMP Exporter (optionnel)

Pour activer le monitoring matériel Synology (températures, RAID, ventilateurs, UPS) :

1. Décommenter le service `snmp-exporter` dans `docker-compose.yml`
2. Dans `config/prometheus.yml`, décommenter le job `synology-snmp` et remplacer `<IP_DU_NAS>` par l'adresse IP réelle du NAS
3. Redéployer la stack

## Structure du projet

```
Monitoring_HomeServer/
├── docker-compose.yml                      # Orchestration des conteneurs
├── config/
│   └── prometheus.yml                      # Configuration des scrape jobs
├── ansible/
│   ├── inventory/
│   │   └── hosts.ini                       # Inventaire (localhost)
│   └── playbooks/
│       └── deploy-monitoring.yml           # Playbook de déploiement
└── .github/
    └── workflows/
        └── cd.yml                          # Pipeline CI/CD
```

## Limites mémoire

| Service | Limite |
|---|---|
| Prometheus | 256 MB |
| Grafana | 256 MB |
| cAdvisor | 256 MB |
| Node Exporter | 32 MB |

Prometheus est configuré avec une rétention de **7 jours** et un plafond de stockage de **512 MB**.

## Accès

| Interface | URL |
|---|---|
| Grafana | `http://<IP_NAS>:3001` |
| Prometheus | `http://<IP_NAS>:9090` |
