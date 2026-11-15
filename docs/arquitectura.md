# Arquitectura · Accesorios que faltan

Las mismas figuras en PNG, para el informe: [local](img/arquitectura-local.png) · [Google Cloud](img/arquitectura-gcp.png).

El mismo código (`ventas-ia/`) corre en dos lugares. Cambia **dónde viven los datos y los modelos** y
**quién levanta los contenedores**, pero no el código: `DATA_URI` y `MODELS_URI` apuntan a una carpeta
en local y a `gs://` en la nube.

## 1. Local: Docker Compose (y k3d para probar Kubernetes)

```mermaid
flowchart LR
    subgraph PC["PC del equipo (WSL)"]
        DATOS[("datos/<br/>Excel anonimizado<br/>(copiado del repo del Curso 1)")]
        MODELOS[("ventas-ia/modelos/<br/>v.../model.joblib<br/>metadata.json<br/>active.json")]

        subgraph COMPOSE["docker compose"]
            TRAINER["trainer<br/>(se corre a mano:<br/>docker compose run trainer)"]
            API["api FastAPI :8000<br/>/predict /healthz /metrics<br/>/versions /seguimiento"]
            PG[("postgres:16<br/>volumen pgdata")]
        end
    end

    USUARIO(["Vendedor / curl / k6"])

    DATOS -->|lee| TRAINER
    TRAINER -->|"guarda versión<br/>y active.json"| MODELOS
    TRAINER -->|trainings<br/>model_versions| PG
    MODELOS -->|carga la versión activa| API
    API -->|"requests, predictions<br/>seguimientos (async)"| PG
    USUARIO -->|HTTP| API
```

Para probar los charts de Helm antes de pagar GKE se usa **k3d** (k3s dentro de Docker), como en la
clase 5: los mismos manifiestos que en la nube, con las imágenes importadas con `make k3d-images` y
los modelos en un volumen local.

## 2. Google Cloud: GKE

```mermaid
flowchart TB
    DEV(["Push / PR en GitHub"])

    subgraph GH["GitHub Actions"]
        CI["ci: tests + build<br/>de api y trainer"]
        CD["deploy: helm upgrade<br/>+ prueba de humo"]
    end

    TF["Terraform / OpenTofu<br/>(desde la PC)"]

    subgraph GCP["Proyecto GCP unlu-ia-deboeuf · us-central1"]
        AR[("Artifact Registry<br/>ventas-api:SHA<br/>ventas-trainer:SHA")]
        GCS[("Cloud Storage<br/>data/ (Excel)<br/>models/v.../<br/>models/active.json")]
        BUDGET["Alerta de presupuesto"]

        subgraph GKE["GKE zonal · 2 nodos e2-standard-2"]
            subgraph NS["namespace ventas"]
                SVC["Service api"]
                APIPODS["Deployment api<br/>1, 3 o 5 réplicas<br/>(HPA aparte, como demo)"]
                JOB["Job trainer<br/>(un mes de corte por corrida)"]
                PGSS[("StatefulSet postgres<br/>+ PVC")]
                K6["Job k6<br/>(pruebas de carga)"]
            end
            subgraph MON["namespace monitoring"]
                PROM["Prometheus"]
                GRAF["Grafana<br/>+ alerta propia"]
            end
        end
    end

    DEV --> CI
    CI -->|OIDC, sin claves| AR
    CI --> CD
    CD -->|OIDC| GKE
    TF -->|crea| AR
    TF -->|crea| GCS
    TF -->|crea| GKE
    TF -->|crea| BUDGET

    AR -.->|imágenes| APIPODS
    AR -.->|imágenes| JOB
    JOB -->|Workload Identity| GCS
    APIPODS -->|Workload Identity<br/>lee active.json| GCS
    JOB --> PGSS
    APIPODS -->|async| PGSS
    K6 --> SVC --> APIPODS
    PROM -->|/metrics| APIPODS
    PROM --> GRAF

    VENDEDOR(["Vendedor / ERP"]) -->|/predict<br/>/seguimiento| SVC
```

## Qué cambia entre los dos

| Pieza | Local | Google Cloud |
|---|---|---|
| Datos de entrenamiento | `datos/` en la PC | `gs://<bucket>/data/` |
| Versiones del modelo | `ventas-ia/modelos/` | `gs://<bucket>/models/` |
| Quién corre los contenedores | Docker Compose (o k3d) | GKE, con charts de Helm |
| Entrenamiento | `docker compose run trainer` | Job de Kubernetes |
| Base de datos | contenedor `postgres:16` con volumen | StatefulSet con PVC (disco persistente) |
| Credenciales | ninguna (todo local) | Workload Identity en pods, OIDC en CI: sin claves JSON |
| Imágenes | se construyen en la PC | CI las publica en Artifact Registry con el SHA |
| Observabilidad | `/metrics` a mano | Prometheus + Grafana |
| Infraestructura | — | Terraform / OpenTofu |

## Estado

| Pieza | Estado |
|---|---|
| API, trainer, Postgres y Docker Compose | Hecho (PR #1) |
| CI: tests y build de las dos imágenes | Hecho |
| Terraform: bucket, Workload Identity, node pool `e2-standard-2`, presupuesto | Pendiente |
| Charts de Helm (api, trainer, postgres) y `deploy.yml` | Pendiente |
| Prometheus/Grafana, k6 | Pendiente |
