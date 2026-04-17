# 🐳 ProxyNginxApache_Enric — Nginx + 2× Apache + Volumen Compartido

## Arquitectura

```
Internet / Host
      │
      ▼  :8080
  ┌─────────────────────────────┐
  │        NGINX                │
  │  · Proxy inverso            │
  │  · Balanceo de carga        │
  │    (least_conn)             │
  │  · Caché de respuestas      │
  └──────────┬──────────────────┘
             │ red interna (web_net)
      ┌──────┴──────┐
      ▼             ▼
 ┌─────────┐  ┌─────────┐
 │ apache1 │  │ apache2 │
 │  :80    │  │  :80    │
 └────┬────┘  └────┬────┘
      │             │
      └──────┬──────┘
             ▼
     📁 Volumen compartido
       ./web  →  /usr/local/apache2/htdocs
       (mismo contenido en los 2 nodos)
```

## Estructura del proyecto

```
docker-practica/
├── docker-compose.yml        # Orquestación de servicios
├── nginx/
│   └── nginx.conf            # Proxy + balanceo + caché
├── apache/
│   └── httpd.conf            # Configuración Apache
├── web/                      # ← Volumen compartido
│   └── index.html            # Página con imágenes, vídeo y cita
└── README.md
```

## Cómo ejecutar

### Requisitos

- [Docker](https://docs.docker.com/get-docker/) ≥ 24
- [Docker Compose](https://docs.docker.com/compose/) v2 (incluido en Docker Desktop)

### Arrancar

```bash
docker compose up -d
```

### Ver la página

Abre el navegador en: **http://localhost:8080**

### Ver logs en tiempo real

```bash
# Todos los servicios
docker compose logs -f

# Solo Nginx (ver balanceo y caché)
docker compose logs -f nginx

# Solo los Apache
docker compose logs -f apache1 apache2
```

### Verificar el balanceo de carga

Las cabeceras de respuesta muestran qué nodo sirvió la petición:

```bash
# Ejecuta varias veces para ver cómo alterna entre apache1 y apache2
curl -I http://localhost:8080
```

Busca la cabecera:
```
X-Upstream-Node: 172.x.x.x:80
X-Cache-Status: MISS   (primera vez)
X-Cache-Status: HIT    (desde caché)
```

### Verificar la caché

```bash
# Primera petición → MISS (va al Apache)
curl -I http://localhost:8080

# Segunda petición → HIT (servida desde caché Nginx)
curl -I http://localhost:8080
```

### Parar los contenedores

```bash
docker compose down
```

### Parar y eliminar volúmenes (limpieza total)

```bash
docker compose down -v
```

## Detalles técnicos

### Nginx — Balanceo de carga

Se usa el algoritmo **`least_conn`** (menor número de conexiones activas), más justo que round-robin cuando las peticiones tienen duraciones variables.

```nginx
upstream apache_cluster {
    least_conn;
    server apache1:80;
    server apache2:80;
    keepalive 32;
}
```

### Nginx — Caché

```nginx
proxy_cache_path /var/cache/nginx
    levels=1:2
    keys_zone=mi_cache:10m
    max_size=500m
    inactive=60m;

proxy_cache            mi_cache;
proxy_cache_valid      200 302  10m;
proxy_cache_use_stale  error timeout ...;
proxy_cache_lock       on;
```

| Parámetro | Valor | Significado |
|---|---|---|
| `keys_zone` | 10 MB | Índice en RAM |
| `max_size` | 500 MB | Máximo en disco |
| `inactive` | 60 min | Se purga si no se usa |
| `proxy_cache_valid 200` | 10 min | TTL respuestas OK |
| `proxy_cache_lock` | on | Sin thundering herd |

### Volumen compartido

Ambos Apache montan **exactamente la misma carpeta** `./web`:

```yaml
volumes:
  web_content:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: ./web
```

Esto garantiza que cualquier cambio en `./web/index.html` se ve en los dos nodos sin reiniciar contenedores.

## Contenido de la página

- **3 imágenes de paisajes** (Alpes, Valle, Lago alpino) — Unsplash
- **Vídeo embebido** de YouTube (paisajes naturales)
- **Cita filosófica** — Lao Tse, Tao Te Ching


> **Nota:** el volumen caché de Nginx (`nginx_cache`) se crea en tiempo de ejecución y no se versiona. El `.gitignore` lo excluye automáticamente al ser un volumen Docker.

