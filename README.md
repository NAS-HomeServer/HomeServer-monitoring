# Monitoring HomeServer

Stack de monitoring pour NAS Synology (Prometheus, Grafana, Alertmanager avec notifications Discord), déployée via Docker Compose et Ansible, avec une pipeline CI/CD GitHub Actions sur runner auto-hébergé.

## Stack

| Service | Image | Port hôte | Rôle | Mémoire max |
|---|---|---|---|---|
| Prometheus | `prom/prometheus:v3.14.0` | 9090 | Base de données de métriques | 512 MB |
| Grafana | `grafana/grafana:13.2.2` | 3001 | Visualisation et dashboards | 512 MB |
| Node Exporter | `prom/node-exporter:v1.12.1` | 9100 | Métriques OS (CPU, RAM, disques, réseau) | 64 MB |
| cAdvisor | `gcr.io/cadvisor/cadvisor:v0.49.2` | — | Métriques des conteneurs Docker | 200 MB |
| Alertmanager | `prom/alertmanager:v0.34.1` | 9093 | Routage des alertes, notifications Discord natives | 96 MB |

Tous les services partagent le réseau bridge `monitoring-homeserver`. Le swap est désactivé (`memswap_limit` = `mem_limit`).

> cAdvisor est volontairement bloqué en `< 0.50.0` dans Renovate (incompatibilité containerd / DSM Synology, cf. incident d'août 2026).

## Prérequis

- NAS Synology avec Container Manager (Docker + Docker Compose v2)
- Répertoire `/volume2/docker` existant
- Runner GitHub Actions auto-hébergé sur le NAS, avec accès au socket Docker de l'hôte
- Secrets GitHub : `MY_GITHUB_TOKEN` (téléchargement du tarball du repo) et `DISCORD_WEBHOOK_URL` (dans l'environnement `production`, passé au playbook Ansible qui dépose le webhook sur le NAS)
- Pour un déploiement manuel : Python 3 + Ansible avec la collection `community.docker`

## Déploiement

### Via GitHub Actions (recommandé)

Le workflow [`pipeline.yml`](.github/workflows/pipeline.yml) se déclenche à chaque push sur `main`, ou manuellement depuis l'onglet **Actions**. Il enchaîne quatre jobs :

| Job | Contenu |
|---|---|
| ① Code Quality | `ansible-lint --profile=production`, `yamllint --strict`, `hadolint` sur le Dockerfile du runner, `promtool check config/rules` et `amtool check-config` |
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
1. Crée les répertoires hôte : `/volume2/docker/prometheus/{config,config/rules,data}` et `grafana/{provisioning/{datasources,dashboards},dashboards/{nas,cyberlab}}` (propriétaire `1026:100`), `/volume2/docker/alertmanager/{config,data,config/secrets}` (propriétaire `65534:65534`, le conteneur tournant en `nobody`)
2. Déploie `config/prometheus.yml`, `config/alertmanager/alertmanager.yml`, le secret webhook Discord (`config/secrets/discord_webhook`, depuis `discord_webhook_url`, jamais versionné) et le provisioning/dashboards Grafana dans ces répertoires
3. Supprime les conteneurs arrêtés **du projet uniquement** (filtre sur le label `com.docker.compose.project`)
4. Déploie la stack via `community.docker.docker_compose_v2` (`pull: missing`, `remove_orphans: true`)
5. Recharge Prometheus et Alertmanager à chaud (`POST /-/reload`, via `docker exec` dans chaque conteneur) et redémarre Grafana, uniquement si le fichier de configuration correspondant a changé

Le conteneur `ansible-runner` n'est pas raccordé au réseau `monitoring` : le rechargement HTTP passe donc par un `docker exec` dans le conteneur cible plutôt que par un appel réseau direct depuis Ansible.

### Via Docker Compose

```bash
docker compose -p monitoring-homeserver up -d
```

> Le webhook Discord n'est pas géré par Docker Compose : déposer manuellement `/volume2/docker/alertmanager/config/secrets/discord_webhook` (contenu = l'URL du webhook, `chmod 400`, owner `65534:65534`) avant de démarrer Alertmanager, ou passer par le playbook Ansible ci-dessus.

## Configuration

### Volumes sur le NAS

| Chemin hôte | Monté dans |
|---|---|
| `/volume2/docker/prometheus/config` | Prometheus — `/etc/prometheus` (`prometheus.yml`, sous-dossier `rules/`) |
| `/volume2/docker/prometheus/data` | Prometheus — `/prometheus` |
| `/volume2/docker/grafana` | Grafana — `/var/lib/grafana` (données internes, non versionné) |
| `/volume2/docker/grafana/provisioning` | Grafana — `/etc/grafana/provisioning` (lecture seule) |
| `/volume2/docker/grafana/dashboards` | Grafana — `/etc/grafana/dashboards` (lecture seule) |
| `/volume2/docker/alertmanager/config` | Alertmanager — `/etc/alertmanager` (`alertmanager.yml`, `secrets/discord_webhook`) |
| `/volume2/docker/alertmanager/data` | Alertmanager — `/alertmanager` |

Le playbook Ansible déploie `config/prometheus.yml`, `config/alertmanager/alertmanager.yml` et le provisioning/dashboards Grafana depuis le repo vers ces répertoires à chaque exécution : le contenu versionné est la source de vérité.

### Prometheus — [`config/prometheus.yml`](config/prometheus.yml)

- Intervalles globaux : `15s` de scrape, `15s` d'évaluation
- Rétention : **30 jours** ou **2 GB** (la première limite atteinte) ; compaction TSDB par défaut (pas de blocs figés)
- Sans lockfile, API `--web.enable-lifecycle` active (rechargement via `POST /-/reload`)
- `rule_files: rules/*.yml` et `alerting` (→ `alertmanager:9093`) configurés ; `config/rules/` est vide pour l'instant (pas encore de règle d'alerte définie)

Jobs configurés :

| Job | Cible | Intervalle |
|---|---|---|
| `synology-nas` | `node-exporter:9100` | 15s |
| `docker-containers` | `cadvisor:8080` | 60s |
| `prometheus` | `localhost:9090` | 15s |

> **Impact rétention 30j/2GB** : la taille reste bornée par `retention.size=2GB`, donc l'usage disque ne grandit pas avec le temps — mais la compaction manipule temporairement plusieurs blocs à la fois et peut ponctuellement utiliser plus de mémoire/disque (~2× brièvement). `mem_limit: 512M` reste probablement suffisant pour le volume de séries actuel (3 exporters, faible cardinalité), mais c'est à surveiller après la mise en prod ; passer à 768M/1G si des OOM apparaissent.

### SNMP (matériel Synology)

Un job `synology-snmp` (températures, RAID, ventilateurs, UPS) a existé dans `prometheus.yml`, mais le service `snmp-exporter` n'a jamais été défini dans `docker-compose.yml` : la cible restait `down` en permanence. Il a été retiré. Pour le réactiver un jour :

1. Ajouter un service `snmp-exporter` (port 9116) sur le réseau `monitoring` dans `docker-compose.yml`
2. Réajouter le job dans `config/prometheus.yml`, avec l'IP réelle du NAS
3. Redéployer la stack

### Alerting

Prometheus envoie ses alertes à Alertmanager (`alertmanager:9093`, configuré via `alerting.alertmanagers`), qui les poste directement sur Discord via le receiver natif `discord_configs` ([`config/alertmanager/alertmanager.yml`](config/alertmanager/alertmanager.yml)). L'alerting intégré de Grafana est désactivé (`GF_ALERTING_ENABLED=false`, `GF_UNIFIED_ALERTING_ENABLED=false`).

Route par défaut : `group_by: [alertname]`, `group_wait: 30s`, `group_interval: 5m`, `repeat_interval: 4h`, `send_resolved: true`.

> L'URL du webhook Discord n'est jamais commitée : le receiver la lit via `webhook_url_file: /etc/alertmanager/secrets/discord_webhook`, un fichier déposé par le playbook Ansible (`mode 0400`, owner `65534:65534`, propriétaire du processus Alertmanager) sur le volume `alertmanager_config`, à partir du secret GitHub `DISCORD_WEBHOOK_URL`.

> Aucune règle d'alerte n'est encore définie (`config/rules/` est vide) : Prometheus n'a rien à évaluer pour l'instant, seule la tuyauterie Prometheus → Alertmanager → Discord est en place.

### Grafana

Fuseau `Europe/Paris`, télémétrie et vérifications de mises à jour désactivées, `GOMAXPROCS=1` pour limiter la consommation CPU sur le NAS.

Datasource et dashboards sont provisionnés depuis git ([`config/grafana/provisioning/`](config/grafana/provisioning/)), en lecture seule dans le conteneur :
- **Datasource** Prometheus (`prometheus`, uid `cfe5m371sd62of`, `http://prometheus:9090`) — nom et uid identiques à la datasource existante, pour ne pas casser les dashboards déjà en place
- **Dashboards** ([`config/grafana/dashboards/`](config/grafana/dashboards/)) organisés en dossiers (`foldersFromFilesStructure: true`) : `nas/` et `cyberlab/`, tous deux vides pour l'instant
- Provisioning non modifiable depuis l'UI (`editable: false`, `allowUiUpdates: false`) : toute évolution passe par git

Un changement de provisioning ou de dashboard entraîne un redémarrage automatique de Grafana (nécessaire pour qu'il relise `/etc/grafana/provisioning` et `/etc/grafana/dashboards`, qu'il ne surveille pas en continu).

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
│   ├── prometheus.yml                  # Scrape jobs, rule_files, alerting
│   ├── rules/                          # Règles d'alerte Prometheus (vide pour l'instant)
│   ├── alertmanager/
│   │   └── alertmanager.yml            # Routage des alertes vers Discord (receiver natif)
│   └── grafana/
│       ├── provisioning/
│       │   ├── datasources/            # Datasource Prometheus provisionnée
│       │   └── dashboards/             # Provider de dashboards (foldersFromFilesStructure)
│       └── dashboards/                 # Dashboards JSON, par dossier (nas/, cyberlab/)
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
