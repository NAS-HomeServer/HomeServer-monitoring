# Monitoring HomeServer

Stack de monitoring pour NAS Synology (Prometheus, Grafana, Alertmanager avec notifications Discord), déployée via Docker Compose et Ansible, avec une pipeline CI/CD GitHub Actions sur runner auto-hébergé.

## Stack

| Service | Image | Port hôte | Rôle | Mémoire max |
|---|---|---|---|---|
| Prometheus | `prom/prometheus:v3.14.0` | 9090 | Base de données de métriques | 512 MB |
| Grafana | `grafana/grafana:13.2.2` | 3001 | Visualisation et dashboards | 512 MB |
| Node Exporter | `prom/node-exporter:v1.12.1` | 9100 | Métriques OS (CPU, RAM, disques, réseau) | 64 MB |
| cAdvisor | `gcr.io/cadvisor/cadvisor:v0.49.2` | — | Métriques des conteneurs Docker | 200 MB |
| Alertmanager | `prom/alertmanager:v0.34.1` | 9093 | Routage des alertes | 96 MB |
| alertmanager-discord | `rogerrum/alertmanager-discord:main` | — | Relais des alertes vers un webhook Discord | 32 MB |

Tous les services partagent le réseau bridge `monitoring-homeserver`. Le swap est désactivé (`memswap_limit` = `mem_limit`).

> cAdvisor est volontairement bloqué en `< 0.50.0` dans Renovate (incompatibilité containerd / DSM Synology, cf. incident d'août 2026).

## Prérequis

- NAS Synology avec Container Manager (Docker + Docker Compose v2)
- Répertoire `/volume2/docker` existant
- Runner GitHub Actions auto-hébergé sur le NAS, avec accès au socket Docker de l'hôte
- Secrets GitHub : `MY_GITHUB_TOKEN` (téléchargement du tarball du repo) et `DISCORD_WEBHOOK_URL` (dans l'environnement `production`)
- Pour un déploiement manuel : Python 3 + Ansible avec la collection `community.docker`

## Déploiement

### Via GitHub Actions (recommandé)

Le workflow [`pipeline.yml`](.github/workflows/pipeline.yml) se déclenche à chaque push sur `main`, ou manuellement depuis l'onglet **Actions**. Il enchaîne quatre jobs :

| Job | Contenu |
|---|---|
| ① Code Quality | `ansible-lint --profile=production`, `yamllint --strict`, `hadolint` sur le Dockerfile du runner |
| ② Security | Vérification Cosign de l'image Trivy, Gitleaks (secrets), Trivy `config` (IaC) avec rapport SARIF en artefact |
| ③ Build & Scan | Build de l'image `ansible-runner`, scan Trivy de l'image, génération d'un SBOM CycloneDX |
| ④ Deploy | Exécution du playbook Ansible depuis l'image construite (gate `environment: production`), puis nettoyage du workspace et de l'image |

Points notables :
- Toutes les images outillage de la CI sont épinglées par digest `sha256`.
- Le token GitHub est passé à `curl` via un fichier de config (jamais en argument de ligne de commande).
- Le workspace (`/volume2/docker/workspace_monitoring`) est nettoyé via le démon Docker de l'hôte, car le conteneur du runner ne voit pas le même montage `/volume2/docker` que le NAS.
- Un finding HIGH/CRITICAL de Trivy bloque le déploiement. Le seul risque accepté est `DS-0002` (runner Ansible en root, nécessaire pour piloter le socket Docker), documenté dans [`.trivyignore`](.trivyignore).

### Via Ansible

```bash
ansible-playbook -i ansible/inventory/hosts.ini ansible/playbooks/deploy-monitoring.yml --extra-vars "discord_webhook_url=<URL_WEBHOOK>"
```

Le playbook attend le `docker-compose.yml` dans `/app` (chemin de l'image `ansible-runner`) ; ajuster `compose_project_dir` pour une exécution hors conteneur.

Le playbook :
1. Crée `/volume2/docker/prometheus/{config,data}` et `/volume2/docker/grafana` (propriétaire `1026:100`)
2. Supprime les conteneurs arrêtés **du projet uniquement** (filtre sur le label `com.docker.compose.project`)
3. Déploie la stack via `community.docker.docker_compose_v2` (`pull: missing`, `remove_orphans: true`)

### Via Docker Compose

```bash
DISCORD_WEBHOOK_URL=<URL_WEBHOOK> docker compose -p monitoring-homeserver up -d
```

## Configuration

### Volumes sur le NAS

| Chemin hôte | Monté dans |
|---|---|
| `/volume2/docker/prometheus/config` | Prometheus — `/etc/prometheus` (doit contenir `prometheus.yml`) |
| `/volume2/docker/prometheus/data` | Prometheus — `/prometheus` |
| `/volume2/docker/grafana` | Grafana — `/var/lib/grafana` |
| `/volume2/docker/alertmanager/config` | Alertmanager — `/etc/alertmanager` (doit contenir `alertmanager.yml`) |
| `/volume2/docker/alertmanager/data` | Alertmanager — `/alertmanager` |

Le playbook ne copie pas les fichiers de configuration : `config/prometheus.yml` et `alertmanager.yml` sont à déposer dans ces répertoires. Les répertoires Alertmanager ne sont pas créés par le playbook.

### Prometheus — [`config/prometheus.yml`](config/prometheus.yml)

- Intervalles globaux : `15s` de scrape, `15s` d'évaluation
- Rétention : **3 jours** ou **256 MB** (la première limite atteinte)
- Blocs TSDB de 2 h, sans lockfile, API `--web.enable-lifecycle` active (rechargement via `POST /-/reload`)

Jobs configurés :

| Job | Cible | Intervalle |
|---|---|---|
| `synology-nas` | `node-exporter:9100` | 15s |
| `docker-containers` | `cadvisor:8080` | 15s |
| `synology-snmp` | `<IP_DU_NAS>` via `snmp-exporter:9116` | 60s |
| `prometheus` | `localhost:9090` | 15s |

### SNMP (matériel Synology)

Le job `synology-snmp` (températures, RAID, ventilateurs, UPS) est présent dans `prometheus.yml`, mais le service `snmp-exporter` n'est pas défini dans `docker-compose.yml`. Pour l'activer :

1. Ajouter un service `snmp-exporter` (port 9116) sur le réseau `monitoring` dans `docker-compose.yml`
2. Remplacer `<IP_DU_NAS>` par l'adresse IP réelle du NAS dans `config/prometheus.yml`
3. Redéployer la stack

Tant que ce n'est pas fait, la cible apparaît en `down` dans Prometheus.

### Alerting

Alertmanager transmet les alertes au conteneur `alertmanager-discord`, qui les poste sur le webhook défini par `DISCORD_WEBHOOK_URL`. L'alerting intégré de Grafana est désactivé (`GF_ALERTING_ENABLED=false`, `GF_UNIFIED_ALERTING_ENABLED=false`).

> `config/prometheus.yml` ne déclare pour l'instant ni `rule_files` ni section `alerting` : Prometheus n'envoie donc aucune alerte à Alertmanager tant que ces blocs ne sont pas ajoutés.

### Grafana

Fuseau `Europe/Paris`, télémétrie et vérifications de mises à jour désactivées, `GOMAXPROCS=1` pour limiter la consommation CPU sur le NAS.

## Sécurité des conteneurs

- `no-new-privileges` sur tous les services
- `cap_drop: ALL` sur tous les services sauf cAdvisor
- Système de fichiers en lecture seule (`read_only`) pour Prometheus, Node Exporter et Alertmanager
- Prometheus et Grafana tournent en non-root (`1026:100`)
- cAdvisor tourne sans `privileged`, avec des montages en lecture seule
- Healthchecks sur Prometheus, Alertmanager et Grafana

## Mises à jour des dépendances

[Renovate](renovate.json) passe le lundi matin (Europe/Paris) :
- images `docker-compose` : review manuelle (digest seul : automerge)
- GitHub Actions : minor/patch groupées en automerge, majors en review
- images épinglées par digest dans le workflow : digest seul en automerge, changement de version en review

## Structure du projet

```
HomeServer-monitoring/
├── docker-compose.yml                  # Orchestration des conteneurs
├── config/
│   └── prometheus.yml                  # Scrape jobs Prometheus
├── ansible/
│   ├── inventory/
│   │   └── hosts.ini                   # Inventaire (localhost)
│   └── playbooks/
│       └── deploy-monitoring.yml       # Playbook de déploiement
├── ci/
│   └── Dockerfile.ansible-runner       # Image Ansible utilisée par la CI
├── .github/
│   └── workflows/
│       └── pipeline.yml                # Pipeline CI/CD (lint, sécurité, build, deploy)
├── renovate.json                       # Mises à jour automatiques des dépendances
├── .trivyignore                        # Risques Trivy acceptés
└── .yamllint.yml                       # Règles yamllint
```

## Accès

| Interface | URL |
|---|---|
| Grafana | `http://<IP_NAS>:3001` |
| Prometheus | `http://<IP_NAS>:9090` |
| Alertmanager | `http://<IP_NAS>:9093` |
| Node Exporter | `http://<IP_NAS>:9100/metrics` |
