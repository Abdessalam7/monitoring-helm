# Datahub V2 Monitoring – Web App Deployment

Stack : Frontend React/Vite (Nginx) + Backend FastAPI déployés sur IKS BNP via Helm.

---

## Structure

```
helm/datahub-v2-web/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── backend-deployment.yaml
    ├── backend-service.yaml
    ├── backend-configmap.yaml
    ├── frontend-deployment.yaml
    ├── frontend-service.yaml
    ├── nginx-configmap.yaml          # Config serveur Nginx (port 8080, proxy /api/)
    ├── nginx-main-configmap.yaml     # Config principale Nginx (pid, temp paths)
    └── ingress.yaml
```

---

## Pré-requis

### 1. Namespace

```bash
kubectl create namespace monitoring-datahub-v2
```

### 2. Pull secret Artifactory

```bash
kubectl create secret docker-registry image-pull-secret \
  --namespace=monitoring-datahub-v2 \
  --docker-server=docker.artifactory-dogen.group.echonet \
  --docker-username=<login> \
  --docker-password=<api-key>
```

### 3. Secret COS (credentials IBM Cloud Object Storage)

```bash
kubectl create secret generic cos-credentials \
  --namespace=monitoring-datahub-v2 \
  --from-literal=COS_ENDPOINT="https://s3.eu-de.cloud-object-storage.appdomain.cloud" \
  --from-literal=COS_API_KEY="<api-key>" \
  --from-literal=COS_INSTANCE_CRN="crn:v1:bluemix:public:..." \
  --from-literal=COS_BUCKET="<bucket-name>"
```

---

## Déploiement

### Dry-run (validation)

```bash
helm upgrade --install datahub-v2-web ./helm/datahub-v2-web \
  --namespace monitoring-datahub-v2 \
  --dry-run
```

### Déploiement réel

```bash
helm upgrade --install datahub-v2-web ./helm/datahub-v2-web \
  --namespace monitoring-datahub-v2
```

### Rollback

```bash
helm rollback datahub-v2-web -n monitoring-datahub-v2
```

### Désinstallation

```bash
helm uninstall datahub-v2-web -n monitoring-datahub-v2
```

---

## Architecture réseau

```
Internet → Ingress (nginx) → frontend Service (ClusterIP:8080)
                                    ↓
                             Nginx (ConfigMap)
                             ├── /          → fichiers React/Vite
                             ├── /assets/   → fichiers statiques (cache 1j)
                             ├── /api/      → proxy → backend:8081
                             └── /health    → 200 OK
                                                ↓
                                     backend Service (ClusterIP:8081)
                                             ↓
                                      FastAPI Pod
                                  (COS credentials via Secret)
```

---

## Build des images

### Backend

Le pipeline GitLab CI build et push automatiquement l'image backend via Kaniko.

### Frontend

Variables CI/CD requises dans GitLab Settings :

| Variable | Description |
|---|---|
| `ARTIFACTORY_TOKEN` | API key Artifactory pour npm |

`.npmrc` utilisé pendant le build :
```
registry=https://repo.artifactory-dogen.group.echonet/artifactory/api/npm/npm/
strict-ssl=false
timeout=3600000
//repo.artifactory-dogen.group.echonet/artifactory/api/npm/npm/:_authToken=${ARTIFACTORY_TOKEN}
```

---

## Vérification post-déploiement

```bash
# État des pods
kubectl get pods -n monitoring-datahub-v2

# Logs backend
kubectl logs -l app=datahub-v2-backend -n monitoring-datahub-v2 --tail=50

# Logs frontend
kubectl logs -l app=datahub-v2-frontend -n monitoring-datahub-v2 --tail=50

# Test health check frontend
kubectl run curl-test --image=curlimages/curl --rm -it --restart=Never \
  -- curl http://datahub-v2-frontend:8080/health

# Test proxy API
kubectl run curl-test --image=curlimages/curl --rm -it --restart=Never \
  -- curl http://datahub-v2-frontend:8080/api/health
```
