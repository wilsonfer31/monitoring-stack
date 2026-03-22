# monitoring-stack

Stack de monitoring générique et réutilisable pour tout projet Docker Compose.

**Loki** · **Promtail** · **Prometheus** · **cAdvisor** · **Node Exporter** · **Grafana**

| Rôle | Outil |
|---|---|
| Collecte des logs Docker | Promtail |
| Stockage des logs | Loki |
| Collecte des métriques containers | cAdvisor |
| Collecte des métriques machine hôte | Node Exporter |
| Stockage des métriques | Prometheus |
| Dashboards | Grafana → http://localhost:3000 |

---

## Structure requise

Ce repo doit être cloné **à côté** du projet que tu veux monitorer :

```
claude/
  mon-projet/                  ← ton projet
    docker-compose.yml
    docker-compose.monitoring.yml   ← référence ../monitoring-stack/
  monitoring-stack/            ← ce repo
    docker-compose.monitoring.yml
    monitoring/
```

---

## Installation

### 1. Cloner les deux repos côte à côte

```bash
git clone https://github.com/wilsonfer31/mon-projet.git
git clone https://github.com/wilsonfer31/monitoring-stack.git
```

### 2. Configurer

Dans le dossier de **ton projet** (pas monitoring-stack), crée un `.env` :

```env
COMPOSE_PROJECT=mon-projet     # nom du projet Docker Compose (= nom du dossier en minuscules)
GRAFANA_PASSWORD=admin         # mot de passe Grafana
```

### 3. Lancer

```bash
cd mon-projet
docker-compose -f docker-compose.yml -f docker-compose.monitoring.yml up -d
```

### 4. Ouvrir Grafana

→ **http://localhost:3000** — login : `admin` / valeur de `GRAFANA_PASSWORD`

Deux dashboards se chargent automatiquement :
- **Logs applicatifs** — logs temps réel avec menu déroulant par container, panneau erreurs filtré, volume par service
- **Métriques système** — RAM/CPU par container (cAdvisor), RAM/disque/charge machine hôte (Node Exporter)

---

## Utiliser avec un nouveau projet

### Option A — fichier `docker-compose.monitoring.yml` dédié (recommandé)

Crée un `docker-compose.monitoring.yml` dans ton projet qui référence ce repo :

```yaml
services:
  loki:
    image: grafana/loki:3.0.0
    container_name: ${COMPOSE_PROJECT}-loki
    command: -config.file=/etc/loki/config.yml
    volumes:
      - ../monitoring-stack/monitoring/loki-config.yml:/etc/loki/config.yml:ro
      - loki_data:/loki
    ports:
      - "3100:3100"
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "wget -qO- http://localhost:3100/ready | grep -q ready"]
      interval: 10s
      timeout: 5s
      retries: 5

  promtail:
    image: grafana/promtail:3.0.0
    container_name: ${COMPOSE_PROJECT}-promtail
    command: -config.file=/etc/promtail/config.yml
    environment:
      COMPOSE_PROJECT: ${COMPOSE_PROJECT}
    volumes:
      - ../monitoring-stack/monitoring/promtail-config.yml:/etc/promtail/config.yml:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
    depends_on:
      loki:
        condition: service_healthy
    restart: unless-stopped

  prometheus:
    image: prom/prometheus:v2.52.0
    container_name: ${COMPOSE_PROJECT}-prometheus
    command:
      - --config.file=/etc/prometheus/config.yml
      - --storage.tsdb.retention.time=30d
    volumes:
      - ../monitoring-stack/monitoring/prometheus-config.yml:/etc/prometheus/config.yml:ro
      - prometheus_data:/prometheus
    ports:
      - "9090:9090"
    restart: unless-stopped

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:v0.49.1
    container_name: ${COMPOSE_PROJECT}-cadvisor
    privileged: true
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
    ports:
      - "8088:8080"
    restart: unless-stopped

  node-exporter:
    image: prom/node-exporter:v1.8.0
    container_name: ${COMPOSE_PROJECT}-node-exporter
    command:
      - --path.rootfs=/host
      - --collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)
    volumes:
      - /:/host:ro
    ports:
      - "9100:9100"
    restart: unless-stopped

  grafana:
    image: grafana/grafana:11.0.0
    container_name: ${COMPOSE_PROJECT}-grafana
    environment:
      GF_SECURITY_ADMIN_PASSWORD: ${GRAFANA_PASSWORD:-admin}
      GF_USERS_ALLOW_SIGN_UP: "false"
    volumes:
      - grafana_data:/var/lib/grafana
      - ../monitoring-stack/monitoring/grafana/provisioning:/etc/grafana/provisioning:ro
    ports:
      - "3000:3000"
    depends_on:
      loki:
        condition: service_healthy
    restart: unless-stopped

volumes:
  loki_data:
  prometheus_data:
  grafana_data:
```

### Option B — lancer directement depuis ce repo

```bash
cd monitoring-stack
cp .env.example .env   # puis éditer COMPOSE_PROJECT
docker-compose -f docker-compose.monitoring.yml up -d
```

---

## Ajouter les métriques de ton backend

Si ton backend expose un endpoint `/metrics`, édite `monitoring/prometheus-config.yml` :

**Spring Boot (Actuator + Micrometer)**
```yaml
- job_name: backend
  metrics_path: /api/actuator/prometheus
  static_configs:
    - targets: ["backend:8080"]
```

**Node.js (prom-client)**
```yaml
- job_name: api
  static_configs:
    - targets: ["api:3000"]
```

**FastAPI**
```yaml
- job_name: api
  static_configs:
    - targets: ["api:8000"]
```

Puis redémarre Prometheus :
```bash
docker restart <compose-project>-prometheus
```

---

## Enrichir les logs avec le contexte utilisateur (optionnel)

Pour voir **qui** a fait quelle requête depuis quel frontend, ajoute dans nginx :

```nginx
proxy_set_header X-Request-Id $request_id;
proxy_set_header X-Client-App "mon-app";
```

Et dans ton backend, injecte ces headers dans les logs (MDC pour Java/Spring, middleware pour Node.js/Express).

---

## Ports utilisés

| Service | Port local |
|---|---|
| Grafana | 3000 |
| Loki | 3100 |
| Prometheus | 9090 |
| cAdvisor | 8088 |
| Node Exporter | 9100 |
