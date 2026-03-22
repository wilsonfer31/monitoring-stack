# monitoring-stack

Stack de monitoring générique pour tout projet Docker Compose.

**Loki** (logs) · **Promtail** (collecte) · **Prometheus** (métriques) · **cAdvisor** (containers) · **Node Exporter** (hôte) · **Grafana** (dashboards)

---

## Démarrage rapide

### 1. Configurer

```bash
cp .env.example .env
```

Éditer `.env` :

```env
# Nom du projet Docker Compose à surveiller (= nom du dossier du projet en minuscules)
COMPOSE_PROJECT=mon-projet

# Mot de passe Grafana
GRAFANA_PASSWORD=admin
```

### 2. Lancer avec ton projet

```bash
docker-compose -f docker-compose.yml -f /chemin/vers/monitoring-stack/docker-compose.monitoring.yml up -d
```

### 3. Ouvrir Grafana

→ **http://localhost:3000** — login : `admin` / valeur de `GRAFANA_PASSWORD`

Deux dashboards sont chargés automatiquement :
- **Logs applicatifs** — logs temps réel, erreurs filtrées, volume par container
- **Métriques système** — RAM/CPU par container, RAM/disque/charge de la machine hôte

---

## Sélectionner les containers dans Grafana

Le dashboard **Logs** a un menu déroulant en haut pour choisir quel(s) container(s) afficher. Tous les containers du projet sont disponibles automatiquement.

---

## Ajouter les métriques de ton backend

Si ton backend expose un endpoint `/metrics` (Prometheus), ajoute-le dans `monitoring/prometheus-config.yml` :

**Spring Boot (Actuator)**
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

**FastAPI (prometheus-fastapi-instrumentator)**
```yaml
- job_name: api
  static_configs:
    - targets: ["api:8000"]
```

Puis redémarre Prometheus :
```bash
docker restart monitoring-prometheus
```

---

## Ports utilisés

| Service | Port |
|---|---|
| Grafana | 3000 |
| Loki | 3100 |
| Prometheus | 9090 |
| cAdvisor | 8088 |
| Node Exporter | 9100 |

---

## Enrichir les logs (optionnel)

Pour avoir le contexte utilisateur dans les logs (qui a fait quoi), ajoute dans ton backend :

**Spring Boot** — injecter `username` et `clientApp` dans le MDC Logback, puis utiliser `LogstashEncoder` en prod.

**Node.js / Express**
```js
const { v4: uuid } = require('uuid');
app.use((req, res, next) => {
  req.requestId = req.headers['x-request-id'] || uuid();
  res.setHeader('x-request-id', req.requestId);
  next();
});
```

**Nginx** — passer le header à ton backend :
```nginx
proxy_set_header X-Request-Id  $request_id;
proxy_set_header X-Client-App  "mon-app";
```
