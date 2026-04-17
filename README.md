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

## Subir a GitHub

```bash
git init
git add .
git commit -m "feat: ProxyNginxApache_Enric - nginx + 2x apache + volumen compartido"
git remote add origin https://github.com/TU_USUARIO/ProxyNginxApache_Enric.git
git push -u origin main
```

> **Nota:** el volumen caché de Nginx (`nginx_cache`) se crea en tiempo de ejecución y no se versiona. El `.gitignore` lo excluye automáticamente al ser un volumen Docker.

---

## Clonar y ejecutar en otro ordenador (paso a paso)

Sigue estos pasos exactos para tener el proyecto funcionando en cualquier máquina desde cero.

### Paso 1 — Instalar Docker

> Si ya tienes Docker instalado, salta al Paso 2.

**Windows / macOS**
1. Ve a https://www.docker.com/products/docker-desktop
2. Descarga **Docker Desktop** para tu sistema operativo
3. Instálalo y ábrelo (necesita estar corriendo en segundo plano)
4. Verifica que funciona abriendo una terminal y ejecutando:
   ```bash
   docker --version
   docker compose version
   ```

**Linux (Ubuntu/Debian)**
```bash
# Instalar Docker
sudo apt update
sudo apt install -y docker.io docker-compose-plugin

# Añadir tu usuario al grupo docker (para no necesitar sudo)
sudo usermod -aG docker $USER

# Cerrar sesión y volver a entrar para que el grupo surta efecto
# Luego verificar:
docker --version
docker compose version
```

---

### Paso 2 — Clonar el repositorio

Abre una terminal y ejecuta:

```bash
git clone https://github.com/TU_USUARIO/ProxyNginxApache_Enric.git
```

Si no tienes Git instalado:
- **Windows:** descarga Git en https://git-scm.com/download/win
- **macOS:** ejecuta `xcode-select --install` en la terminal
- **Linux:** `sudo apt install -y git`

---

### Paso 3 — Entrar en la carpeta del proyecto

```bash
cd ProxyNginxApache_Enric
```

La estructura que deberías ver:

```
ProxyNginxApache_Enric/
├── docker-compose.yml
├── nginx/
│   └── nginx.conf
├── apache/
│   └── httpd.conf
├── web/
│   └── index.html
├── .gitignore
└── README.md
```

---

### Paso 4 — Levantar los contenedores

```bash
docker compose up -d
```

Este comando:
1. Descarga las imágenes de Nginx y Apache si no las tienes (solo la primera vez)
2. Crea la red interna `web_net`
3. Crea el volumen compartido montando `./web`
4. Arranca los 3 contenedores en segundo plano (`-d` = detached)

Verifica que los 3 contenedores están corriendo:

```bash
docker compose ps
```

Deberías ver algo así:

```
NAME           IMAGE          STATUS          PORTS
nginx_proxy    nginx:alpine   Up (healthy)    0.0.0.0:8080->80/tcp
apache1        httpd:alpine   Up (healthy)    80/tcp
apache2        httpd:alpine   Up (healthy)    80/tcp
```

---

### Paso 5 — Abrir la página web

Abre el navegador y ve a:

```
http://localhost:8080
```

Verás la página con las imágenes de paisajes, el vídeo y la cita filosófica.

---

### Paso 6 — Comprobar que el balanceo y la caché funcionan

Abre una terminal y ejecuta varias veces seguidas:

```bash
curl -I http://localhost:8080
```

Fíjate en estas cabeceras de respuesta:

| Cabecera | Qué indica |
|---|---|
| `X-Upstream-Node` | IP del Apache que respondió (cambia entre peticiones) |
| `X-Cache-Status: MISS` | Nginx fue al Apache a buscar el contenido |
| `X-Cache-Status: HIT` | Nginx lo sirvió desde su caché (más rápido) |
| `X-Served-By: Nginx-LoadBalancer` | Confirmación de que pasa por Nginx |

---

### Paso 7 — Ver los logs

```bash
# Ver logs de todos los servicios en tiempo real
docker compose logs -f

# Solo Nginx (muestra balanceo y estado de caché)
docker compose logs -f nginx

# Solo los Apache
docker compose logs -f apache1 apache2
```

Pulsa `Ctrl + C` para salir de los logs (los contenedores siguen corriendo).

---

### Parar el proyecto

```bash
# Parar los contenedores (se pueden volver a arrancar con "up -d")
docker compose down

# Parar y eliminar también los volúmenes (limpieza total)
docker compose down -v
```

---

### Solución de problemas frecuentes

**El puerto 8080 ya está en uso**
```bash
# Ver qué proceso lo está usando
# Windows (PowerShell):
netstat -ano | findstr :8080

# macOS / Linux:
lsof -i :8080

# Solución: cambiar el puerto en docker-compose.yml
ports:
  - "9090:80"   # usar 9090 en vez de 8080
```

**"Cannot connect to the Docker daemon"**
→ Docker Desktop no está abierto. Ábrelo y espera a que el icono de la ballena aparezca en la barra de tareas.

**Los contenedores aparecen como "Exited"**
```bash
# Ver el error concreto
docker compose logs nginx
docker compose logs apache1
```
