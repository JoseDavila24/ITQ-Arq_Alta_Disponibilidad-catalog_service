---
Maestro: IVONNE TATIANA ALCANTARA LLANAS
---
# Reporte de Laboratorio 1.3: Despliegue en Kubernetes

📄 **Reporte formal (PDF en Google Drive):** [Abrir reporte del Lab 1.3](https://drive.google.com/file/d/1LX9U26GhOGiUgZDrUVCpQVQI_DKymnCv/view?usp=sharing)
🐳 **Imagen en Docker Hub:** [`22140248/catalog-service:1.0.0`](https://hub.docker.com/repository/docker/22140248/catalog-service/general)

## 1. Datos generales

| Campo | Valor |
|---|---|
| **Laboratorio** | Clase 1.3: Despliegue de `catalog-service` en Kubernetes |
| **Servicio** | `catalog-service` (Quarkus 3.38.0, OpenJDK 25) |
| **Imagen** | `docker.io/22140248/catalog-service:1.0.0` |
| **Herramientas** | Maven, Podman, Docker Hub, kind, kubectl |
| **Clúster** | kind, nodo `lab-k8s-control-plane` |
| **Entorno** | Fedora (`chema@fedora`) |
| **Evidencias** | [`terminal1`](terminal1) (build, imagen y despliegue) y [`terminal2`](terminal2) (pruebas y `describe`) |

## 2. Objetivo

Llevar el servicio que en los laboratorios 1.1 y 1.2 corría en modo dev a un entorno orquestado:

1. Empaquetar la aplicación y construir una imagen de contenedor.
2. Publicar la imagen en un registro público (Docker Hub).
3. Desplegarla en Kubernetes con un `Deployment` y un `Service`.
4. Hacer que Kubernetes valide por sí mismo los endpoints `/ready` y `/live`, que en el Lab 1.1 se revisaron a mano con `curl`.

## 3. Procedimiento

### 3.1 Compilación con Maven

Se empaquetó la aplicación (`target/quarkus-app/`) y el build terminó con `BUILD SUCCESS` en 53.5 s.

### 3.2 Construcción de la imagen con Podman

```bash
podman build -t docker.io/22140248/catalog-service:1.0.0 .
```

El `Dockerfile` parte de `registry.access.redhat.com/ubi9/openjdk-25-runtime:1.24`, copia las capas de `quarkus-app`, expone el puerto 8080 y se ejecuta con el usuario sin privilegios `185`. La imagen quedó en **484 MB** (ID `9ef9fe52cbb9`).

### 3.3 Publicación en Docker Hub y comprobación

```bash
podman login docker.io
podman push docker.io/22140248/catalog-service:1.0.0
podman image rm docker.io/22140248/catalog-service:1.0.0
podman pull docker.io/22140248/catalog-service:1.0.0
```

Después del `push` se borró la imagen local y se volvió a descargar desde Docker Hub. El ID obtenido fue el mismo (`9ef9fe52cbb9`), así que la imagen publicada es la que se construyó.

### 3.4 Despliegue en Kubernetes

En [`k8s/01-deployment.yaml`](../../k8s/01-deployment.yaml) se cambió el marcador `<tu-usuario-dockerhub>` por `22140248`. Luego se aplicaron los manifiestos:

```bash
kubectl apply -f k8s/01-deployment.yaml
kubectl apply -f k8s/02-service.yaml
```

| Recurso | Configuración principal |
|---|---|
| `Deployment` | 1 réplica, imagen `22140248/catalog-service:1.0.0`, puerto 8080 |
| `readinessProbe` | `GET /ready`, delay 5 s, cada 5 s, timeout 2 s, `failureThreshold` 2 |
| `livenessProbe` | `GET /live`, delay 10 s, cada 10 s, timeout 2 s, `failureThreshold` 3 |
| `Service` | `ClusterIP`, puerto 8080 → `targetPort` 8080 |

### 3.5 Acceso y pruebas

```bash
kubectl port-forward svc/catalog-service 8080:8080
curl -s http://localhost:8080/v1/products | jq .
curl -s http://localhost:8080/health | jq .
curl -s http://localhost:8080/ready | jq .
curl -s http://localhost:8080/live | jq .
kubectl describe pod -l app=catalog-service
```

## 4. Resultados

| Verificación | Resultado |
|---|---|
| `kubectl get deployment` | `READY 1/1`, `AVAILABLE 1` |
| `kubectl get pods` | `catalog-service-6bb88c8d76-vk76f`: `1/1 Running`, **0 reinicios** |
| `kubectl get service` | `ClusterIP 10.96.124.39`, puerto `8080/TCP` |
| `GET /v1/products` | `{"status":"SUCCESS","data":["Product A","Product B","Product C"]}` |
| `GET /health` | `UP` (`catalog-database-check: UP`) |
| `GET /ready` | `UP` (`catalog-database-check: UP`) |
| `GET /live` | `UP` |
| Imagen descargada por el nodo | `Successfully pulled image "22140248/catalog-service:1.0.0" in 10.981s` (179 MB comprimida) |
| Condiciones del pod | `Ready`, `ContainersReady`, `Initialized` y `PodScheduled` en `True` |

### Observación: readiness probe con 503

En los eventos del pod aparece:

```text
Warning  Unhealthy  2m46s (x2 over 3m36s)  kubelet  Readiness probe failed: HTTP probe failed with statuscode: 503
```

Es lo que se esperaba. `ReadinessHealthCheck` simula un clúster de base de datos que falla el **5 %** de las veces y en ese caso responde `DOWN` (HTTP 503). Kubernetes detectó esas dos fallas aisladas. Como `failureThreshold` es 2 y las fallas no fueron consecutivas, el pod siguió `Ready` y nunca salió del `Service`. La *liveness probe* no falló ni una vez, por eso el contador de reinicios quedó en 0.

## 5. Conclusión

El `catalog-service` quedó empaquetado como imagen inmutable, publicado en Docker Hub y desplegado en un clúster de Kubernetes. Los health checks que en el Lab 1.1 se revisaban a mano ahora los ejecuta el `kubelet` de forma continua:

- **Readiness** decide si el pod recibe tráfico. Así se vio con las fallas simuladas de `/ready`: el servicio no se marcó como no disponible por una falla aislada.
- **Liveness** decide si el contenedor se reinicia. No hizo falta ningún reinicio.

Con esto el servicio queda listo para los siguientes pasos de alta disponibilidad: más réplicas, *rolling updates* y observabilidad.
