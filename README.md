# Proyecto final: Nginx + backends + monitoreo con Docker

Infraestructura en contenedores Docker: un **Nginx** que reparte el tráfico entre **3 backends** (Flask), y monitoreo con **Prometheus**, **Grafana**, **nginx-exporter** y **cAdvisor**. Incluye pruebas de carga con **Artillery**.

## Cómo funciona

```
El navegador (o Artillery) entra por Nginx, en el puerto 8080. Nginx reparte entre app1, app2 y app3, cada 5 segundos Prometheus lee las métricas de los backends, de Nginx (a través de un exporter) y de cAdvisor, y Grafana le pide los datos a Prometheus. Todos los contenedores están en una red Docker propia, proyecto_red.
```

## Requisitos previos

- Docker y Docker Compose v2 (`docker compose version` debe funcionar).
- Unos 4 GB de memoria libres para Docker.
- Puertos libres en el equipo: **8080**, **8081**, **9090** y **3000**. Si alguno está ocupado, cámbialo en el archivo `.env` (hay líneas comentadas de ejemplo).

## Estructura del proyecto

```
proyecto-final/
├── docker-compose.yml          # backends, Nginx, Artillery, red y volúmenes
├── .env                        # carga los 4 archivos de Compose y permite cambiar puertos
├── backend/                    # app Flask (Dockerfile, app.py, requirements.txt)
├── nginx/nginx.conf            # proxy inverso, balanceo y stub_status
├── monitoring/
│   ├── part1.yml               # cAdvisor
│   ├── part2.yml               # nginx-exporter y Prometheus
│   ├── part3.yml               # Grafana
│   ├── prometheus/prometheus.yml
│   └── grafana/                # fuente de datos y dashboard de ejemplo
├── performance-test.yml        # prueba de carga normal
└── performance-test-slow.yml   # prueba de carga con backend lento
```

## Cómo levantarlo

```bash
git clone <URL-DEL-REPOSITORIO>
cd proyecto-final
docker compose up -d --build
docker compose ps
```

La primera vez tarda un poco porque construye la imagen del backend y descarga las demás. Gracias al `.env`, un solo comando levanta todo (no hace falta usar `-f`). Cuando `docker compose ps` muestre los servicios en `running` (los backends en `healthy`), ya está listo.

## URLs

| Servicio | URL |
|---|---|
| Aplicación (Nginx) | http://localhost:8080 |
| Prometheus | http://localhost:9090 |
| Grafana (sin usuario ni contraseña) | http://localhost:3000 |
| cAdvisor | http://localhost:8081 |

## Verificar el balanceo

1. Abre http://localhost:8080 y recarga varias veces: debe salir `Hello from server 1`, luego `server 2`, luego `server 3` y vuelve a empezar.
2. O desde la terminal:

   ```bash
   # Linux / macOS / WSL
   for i in 1 2 3 4 5 6; do curl -s localhost:8080 | grep Hello; done
   ```

   ```powershell
   # PowerShell
   1..6 | ForEach-Object { curl.exe -s localhost:8080 }
   ```

3. Mira a qué backend fue cada petición en los logs de Nginx (campo `upstream=`):

   ```bash
   docker compose logs nginx --tail 10
   ```

## Verificar el monitoreo

1. En Prometheus (http://localhost:9090) entra a **Status → Target health**. Deben aparecer 6 targets en **UP**: `app1`, `app2`, `app3`, `nginx-exporter`, `cadvisor` y `prometheus`.
2. En la pestaña **Query** prueba `up`: todo debe dar `1`.
3. En Grafana (http://localhost:3000) la fuente de datos Prometheus ya viene configurada. Para el dashboard:
   - Importa el de ejemplo: **Dashboards → New → Import → Upload dashboard JSON file** y elige `monitoring/grafana/solucion/lab-proxy.json`.
   - O crea tus propios paneles con estas consultas:

| Qué muestra | Consulta |
|---|---|
| Conexiones activas en Nginx | `nginx_connections_active` |
| Peticiones por backend | `sum by (backend) (rate(flask_http_request_total[30s]))` |
| Latencia p95 por backend | `histogram_quantile(0.95, sum by (le, backend) (rate(flask_http_request_duration_seconds_bucket[30s])))` |
| CPU | `rate(container_cpu_usage_seconds_total[30s])` |
| Memoria | `container_memory_usage_bytes` |
| Estado de los servicios | `up` |

## Pruebas de carga

Se hacen con Artillery, que corre en un contenedor (perfil `test`, por eso no arranca con `up`). Abre el dashboard de Grafana en **Last 15 minutes** con refresco de 5 s y, en otra terminal, dentro de la carpeta del proyecto:

```bash
# Prueba normal (unos 90 s): sube de 5 a 50 usuarios por segundo y se queda en 50
docker compose run --rm artillery

# Prueba con backend lento (unos 60 s): 50 usuarios por segundo pidiendo /slow?ms=500
docker compose run --rm artillery run /scripts/performance-test-slow.yml
```

Al final Artillery imprime un **Summary report** con las peticiones, los códigos de respuesta y los tiempos (mediana, p95, p99).

Resultados que obtuvimos:

| | Normal | Backend lento |
|---|---|---|
| Peticiones | 3.825 | 3.000 |
| Respuestas 200 | 3.825 | 3.000 |
| Fallidas | 0 | 0 |
| Tiempo promedio | 1,2 ms | 1.848 ms |
| p95 | 2 ms | 3.072 ms |
| Máximo | 16 ms | 3.349 ms |

## Apagar todo

```bash
docker compose down        # apaga y borra los contenedores (conserva los datos)
docker compose down -v     # además borra los volúmenes (métricas y dashboards)
```

## Problemas comunes

- **`no configuration file provided: not found`:** no estás dentro de la carpeta del proyecto. Entra con `cd` y repite el comando.
- **Un puerto está ocupado o `forbidden by its access permissions` (Windows):** cambia el puerto en `.env` (por ejemplo `GRAFANA_PORT=3001`) y vuelve a ejecutar `docker compose up -d`. Si es el 8080 de Nginx, cámbialo en `docker-compose.yml`.
- **Nginx no arranca:** espera a que los backends estén `healthy` (`docker compose ps`) y mira `docker compose logs nginx`.
- **cAdvisor no muestra métricas por contenedor (Docker Desktop con WSL):** solo entrega las de la máquina completa. En ese caso usa `docker stats` para ver cada contenedor.

## Notas

- Grafana está sin contraseña y con rol Admin **solo para pruebas locales**.
- Prometheus lee cada 5 s para ver los cambios rápido; en un servidor real se usaría 15 s o más.
