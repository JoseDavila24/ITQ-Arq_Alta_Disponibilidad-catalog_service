# Catalog Service - Unidad 1

**Servicio de catálogo** para la plataforma e-commerce. Forma parte del proyecto integrador de **Arquitectura de despliegue de alta disponibilidad**.

> [!IMPORTANT]
> **Maestra, por favor revise la carpeta [`resultados/`](resultados/).**
> Ahí están los **logs** de cada ejecución y los **análisis (reportes)** de cada modificación que se le hizo al servicio, ordenados por laboratorio.

---

## 📈 Avance de la práctica

| Laboratorio | Tema | Estado | Evidencia | Reporte general (PDF) |
|---|---|---|---|---|
| **1.1 y 1.2** | Línea base de disponibilidad y patrones de resiliencia | ✅ Terminado | [`resultados/lab 1-1 y 1-2/`](resultados/lab%201-1%20y%201-2/) | [Ver en Google Drive](https://drive.google.com/file/d/1y8-bnx1Fnwae2CYFohO4SeKCTGGPxo3x/view?usp=sharing) |
| **1.3** | Despliegue en Kubernetes | ⏳ Pendiente | *Próximamente* | *Próximamente* |

---

## 🧪 Laboratorio 1.1 y 1.2: Disponibilidad y resiliencia

📄 **Reporte general (PDF en Google Drive):** [Abrir reporte del Lab 1.1 y 1.2](https://drive.google.com/file/d/1y8-bnx1Fnwae2CYFohO4SeKCTGGPxo3x/view?usp=sharing)

📁 **Carpeta de evidencias:** [`resultados/lab 1-1 y 1-2/`](resultados/lab%201-1%20y%201-2/)

### Contenido de la carpeta

```text
resultados/lab 1-1 y 1-2/
├── Logs/                       # Salida de cada ejecución de test-availability.sh (200 solicitudes)
│   ├── availability-baseline.txt
│   ├── availability-modification1.txt
│   ├── availability-modification2.txt
│   ├── availability-modification3.txt
│   └── resilience-sample.txt
└── Reportes individuales/      # Análisis de cada etapa + capturas de logs del servidor
    ├── Reporte de Laboratorio 1.1 Línea Base de Disponibilidad.md
    ├── Reporte de Laboratorio 1.2 Etapa 1.md
    ├── Reporte de Laboratorio 1.2 Etapa 2.md
    ├── Reporte de Laboratorio 1.2 Etapa 3.md
    └── Fragmento de logs del servidor m1.png / m2.png / m3.png
```

### Resumen por modificación

| Etapa | Cambio aplicado | Log | Reporte | Disponibilidad observada |
|---|---|---|---|---|
| Lab 1.1: Línea base | Sin patrones de resiliencia | [`availability-baseline.txt`](resultados/lab%201-1%20y%201-2/Logs/availability-baseline.txt) | [Reporte 1.1](resultados/lab%201-1%20y%201-2/Reportes%20individuales/Reporte%20de%20Laboratorio%201.1%20L%C3%ADnea%20Base%20de%20Disponibilidad.md) | **65.50 %** |
| Lab 1.2: Etapa 1 | `@Timeout(800)` + `@Retry(maxRetries = 2, delay = 150)` | [`availability-modification1.txt`](resultados/lab%201-1%20y%201-2/Logs/availability-modification1.txt) | [Reporte Etapa 1](resultados/lab%201-1%20y%201-2/Reportes%20individuales/Reporte%20de%20Laboratorio%201.2%20Etapa%201.md) | **43.50 %** |
| Lab 1.2: Etapa 2 | + `@Fallback(fallbackMethod = "getCatalogFallback")` | [`availability-modification2.txt`](resultados/lab%201-1%20y%201-2/Logs/availability-modification2.txt) | [Reporte Etapa 2](resultados/lab%201-1%20y%201-2/Reportes%20individuales/Reporte%20de%20Laboratorio%201.2%20Etapa%202.md) | **100.00 %** |
| Lab 1.2: Etapa 3 | + `@CircuitBreaker(requestVolumeThreshold = 10, failureRatio = 0.7, delay = 5000)` | [`availability-modification3.txt`](resultados/lab%201-1%20y%201-2/Logs/availability-modification3.txt) | [Reporte Etapa 3](resultados/lab%201-1%20y%201-2/Reportes%20individuales/Reporte%20de%20Laboratorio%201.2%20Etapa%203.md) | **100.00 %** |

Además, [`resilience-sample.txt`](resultados/lab%201-1%20y%201-2/Logs/resilience-sample.txt) contiene una muestra adicional con todos los patrones activos.

> Formato de cada línea de log: `NNN <código_http> <tiempo_total> <cuerpo_respuesta>`.
> Las respuestas `DEGRADED_CACHE` son HTTP 200 servidas por el fallback (datos en caché).

---

## 🧪 Laboratorio 1.3: Despliegue en Kubernetes

⏳ **Pendiente.** Cuando esté terminado, aquí se agregarán:

- 📁 Carpeta de evidencias: `resultados/lab 1-3/`
- 📄 Reporte general (PDF en Google Drive): *enlace pendiente*

---

## 📋 Descripción del servicio

Microservicio base que simula latencia, fallas y health checks. Los patrones de resiliencia (Timeout, Retry, Circuit Breaker, Fallback) se implementan de forma progresiva en los laboratorios.

## 🛠️ Stack

- **Java**: OpenJDK 25
- **Framework**: Quarkus 3.38.0
- **Build**: Maven 3.9.6

## 🚀 Ejecución rápida

```bash
./mvnw quarkus:dev
```

Endpoints:

- `http://localhost:8080/v1/products`: Catálogo con simulación de latencia y fallas
- `http://localhost:8080/health`: Health check
- `http://localhost:8080/ready`: Readiness probe
- `http://localhost:8080/live`: Liveness probe

## 📂 Estructura del repositorio

```text
catalog-service/
├── src/main/java/com/ecom/catalog/   # Código fuente (CatalogResource, ReadinessHealthCheck)
├── src/main/resources/               # application.properties
├── k8s/                              # Manifiestos de Kubernetes (Lab 1.3)
├── resultados/                       # 👈 Logs y análisis de cada laboratorio
├── Dockerfile
└── pom.xml
```

## 📚 Laboratorios del curso

1. **Laboratorio 1.1**: Línea base de disponibilidad
2. **Laboratorio 1.2**: Patrones de resiliencia básicos
3. **Laboratorio 1.3**: Despliegue en Kubernetes
4. **Unidad 5**: Integrar observabilidad con Prometheus, Grafana y OpenTelemetry

## 🤝 Contribución

Este proyecto es parte del material educativo de la asignatura. Para sugerencias o mejoras, contacta al instructor.

## 📄 Licencia

Material educativo - Instituto Tecnológico de Querétaro © 2026
