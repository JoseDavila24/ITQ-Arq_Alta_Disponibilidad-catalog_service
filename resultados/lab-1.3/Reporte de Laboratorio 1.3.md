---
Maestro: IVONNE TATIANA ALCANTARA LLANAS
---
# Reporte de Laboratorio 1.3: De Docker a Kubernetes Local

📄 **Reporte formal (PDF en Google Drive):** [Abrir reporte del Lab 1.3](https://drive.google.com/file/d/1LX9U26GhOGiUgZDrUVCpQVQI_DKymnCv/view?usp=sharing)
🐳 **Imagen en Docker Hub:** [`22140248/catalog-service:1.0.0`](https://hub.docker.com/repository/docker/22140248/catalog-service/general)

## 1. Datos generales

| Campo | Valor |
|---|---|
| **Laboratorio** | Clase 1.3: De Docker a Kubernetes Local |
| **Servicio** | `catalog-service` resiliente del Lab 1.2 (`@Timeout`, `@Retry`, `@CircuitBreaker`, `@Fallback`) |
| **Imagen** | `docker.io/22140248/catalog-service:1.0.0` |
| **Motor de contenedores** | Podman (compatible con los comandos de Docker de la guía) |
| **Kubernetes local** | Kind, nodo `lab-k8s-control-plane` |
| **Entorno** | Fedora, OpenJDK 25, Quarkus 3.38.0 |
| **Evidencias** | [`terminal1`](terminal1) (build, imagen y despliegue) y [`terminal2`](terminal2) (pruebas con `curl` y `kubectl describe pod`). La limpieza (Paso 5) se registra solo con los comandos |

## 2. Objetivo

Empaquetar el `catalog-service` del Lab 1.2 en una imagen de contenedor, publicarla en Docker Hub y desplegarla en un clúster Kubernetes local con archivos YAML. La idea es ver la diferencia entre ejecutar un contenedor de forma **imperativa** (`docker run`) y **declararlo** en Kubernetes, donde el clúster además valida los health checks automáticamente.

> **Nota sobre Podman:** la guía usa `docker`; en Fedora se usó `podman`, que acepta los mismos comandos (`build`, `login`, `push`, `image rm`, `pull`). Por eso las imágenes llevan el prefijo explícito `docker.io/`.

## 3. Procedimiento

### Paso 1: Kubernetes local

El clúster Kind ya estaba corriendo como contenedor (`podman ps` muestra `lab-k8s-control-plane` en `127.0.0.1:45577->6443`), y `kubectl` estaba conectado a él.

### Paso 2: Construir la imagen y subirla a Docker Hub

```bash
./mvnw clean package -DskipTests                           # BUILD SUCCESS (53.5 s)
podman build -t docker.io/22140248/catalog-service:1.0.0 .  # 11 pasos, imagen 9ef9fe52cbb9 (484 MB)
podman login docker.io                                      # Login Succeeded!
podman push docker.io/22140248/catalog-service:1.0.0        # Writing manifest to image destination
podman image rm docker.io/22140248/catalog-service:1.0.0    # Untagged + Deleted
podman pull docker.io/22140248/catalog-service:1.0.0        # Se descarga de nuevo: 9ef9fe52cbb9
```

La imagen descargada de Docker Hub tiene el **mismo ID** (`9ef9fe52cbb9`) que la construida localmente, así que lo publicado es exactamente lo que se compiló.

### Paso 3: Despliegue con YAML

En [`k8s/01-deployment.yaml`](../../k8s/01-deployment.yaml) se cambió el campo `image`:

```yaml
image: 22140248/catalog-service:1.0.0
imagePullPolicy: IfNotPresent
```

[`k8s/02-service.yaml`](../../k8s/02-service.yaml) se dejó sin cambios (`ClusterIP`, puerto 8080). Luego:

```bash
kubectl apply -f k8s/01-deployment.yaml   # deployment.apps/catalog-service created
kubectl apply -f k8s/02-service.yaml      # service/catalog-service created
```

| Probe | Endpoint | Configuración |
|---|---|---|
| `readinessProbe` | `GET /ready` | delay 5 s, cada 5 s, timeout 2 s, `failureThreshold` 2 |
| `livenessProbe` | `GET /live` | delay 10 s, cada 10 s, timeout 2 s, `failureThreshold` 3 |

### Paso 4: Validar funcionamiento

```bash
kubectl get deployment catalog-service
kubectl get pods -l app=catalog-service
kubectl get service catalog-service
kubectl port-forward svc/catalog-service 8080:8080
curl -s http://localhost:8080/v1/products | jq .   # y /health, /ready, /live
kubectl describe pod -l app=catalog-service
```

### Paso 5: Limpieza

Al terminar las pruebas se liberaron todos los recursos:

```bash
# Ctrl + C en la terminal del port-forward para cerrar el túnel
kubectl delete deployment catalog-service   # Elimina el Deployment y su pod
kubectl delete service catalog-service      # Elimina el Service
kind delete cluster --name lab-k8s          # Elimina el clúster Kind completo
podman logout docker.io                     # Cierra la sesión de Docker Hub
```

Además de lo que pide la guía, se eliminó el clúster Kind para liberar recursos de la máquina y se cerró la sesión en Docker Hub para no dejar credenciales guardadas. La imagen no se pierde porque sigue publicada en Docker Hub, y los manifiestos de `k8s/` permiten volver a desplegarla igual en la Unidad 2.

> Las salidas de estos comandos no se guardaron; aquí solo se registran los comandos ejecutados.

## 4. Criterios de aceptación

| # | Criterio de la guía | Evidencia | Estado |
|---|---|---|---|
| 1 | Imagen construida y publicada en Docker Hub | `podman build` + `podman push` en `terminal1`; tag `1.0.0` en [Docker Hub](https://hub.docker.com/repository/docker/22140248/catalog-service/general) | ✅ |
| 2 | YAML actualizado con mi repositorio | `image: 22140248/catalog-service:1.0.0` en `k8s/01-deployment.yaml` | ✅ |
| 3 | Pod `Running` y `READY 1/1`, descargado desde Docker Hub | `catalog-service-6bb88c8d76-vk76f 1/1 Running 0`; eventos `Pulling` → `Successfully pulled image "22140248/catalog-service:1.0.0" in 10.981s` → `Created` → `Started` | ✅ |
| 3 | Health checks validados automáticamente | `kubectl describe` muestra las probes `Liveness` y `Readiness` configuradas, condición `Ready: True`, `Restart Count: 0` | ✅ (ver nota 5.1) |
| 4 | `/v1/products` responde `SUCCESS` o `DEGRADED_CACHE` | `{"status":"SUCCESS","data":["Product A","Product B","Product C"]}` | ✅ |
| 4 | `/health`, `/ready`, `/live` responden `UP` | Los tres con `"status": "UP"` en `terminal2` | ✅ |

Resultados de `kubectl get`:

```text
deployment  catalog-service   READY 1/1   UP-TO-DATE 1   AVAILABLE 1
pod         catalog-service-6bb88c8d76-vk76f   1/1   Running   RESTARTS 0
service     catalog-service   ClusterIP   10.96.124.39   8080/TCP
```

## 5. Observaciones

### 5.1 No aparecen eventos "Readiness probe succeeded"

La guía sugiere buscar líneas como `Readiness probe succeeded` en los eventos. En la práctica, el `kubelet` **solo registra eventos cuando una probe falla**; los éxitos no generan eventos. Por eso la prueba de que las probes se ejecutan con éxito está en el estado del pod:

- `Ready: True` y `ContainersReady: True`, que solo se alcanzan si la `readinessProbe` responde bien.
- `Restart Count: 0`, que indica que la `livenessProbe` nunca falló 3 veces seguidas.

### 5.2 Dos fallas de readiness (HTTP 503)

```text
Warning  Unhealthy  2m46s (x2 over 3m36s)  kubelet  Readiness probe failed: HTTP probe failed with statuscode: 503
```

Estas fallas son esperadas: `ReadinessHealthCheck` simula que la base de datos falla el **5 %** de las veces y en ese caso responde `DOWN` (503). Que aparezcan es justamente la prueba de que Kubernetes revisa `/ready` por sí solo. Como fueron fallas aisladas (no 2 seguidas, que es el `failureThreshold`), el pod nunca salió del `Service`.

## 6. Explicación técnica integrada

Compilé el servicio con Maven y construí con Podman la imagen `22140248/catalog-service:1.0.0`. La subí a Docker Hub y, para comprobar que se podía descargar desde cualquier equipo, la borré localmente y la volví a bajar con el mismo ID. Después declaré el despliegue en `k8s/01-deployment.yaml`: qué imagen usar, desde mi repositorio de Docker Hub, y qué health checks ejecutar. Al aplicar el YAML, Kubernetes descargó la imagen (evento `Successfully pulled image`), creó el contenedor y comenzó a validar automáticamente `/ready` cada 5 segundos y `/live` cada 10. En el Lab 1.1 yo revisaba esos endpoints a mano con `curl`; ahora es el clúster quien lo hace. Si `/ready` falla, el pod deja de recibir tráfico; si `/live` falla, el contenedor se reinicia. Las dos fallas simuladas de `/ready` mostraron este control en acción, sin afectar el servicio. El flujo completo es: build local → push al registry → deploy declarativo → validación automática por Kubernetes.

## 7. Conclusión

La aplicación es la misma del Lab 1.2, con sus patrones de resiliencia; lo que cambió es cómo se ejecuta. En lugar de un `docker run` manual, un archivo YAML describe el estado deseado y Kubernetes se encarga de descargar la imagen, arrancarla y vigilar su salud. Estos manifiestos y esta imagen son la base para la Unidad 2, donde se agregarán réplicas, escalamiento automático y configuración externa.
