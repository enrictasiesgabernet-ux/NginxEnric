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

## Configuració de Nginx

### Què hem fet

Nginx és el punt d'entrada de totes les peticions. Hem configurat tres coses:

**Balanceig de càrrega** amb `least_conn`, que envia cada petició al servidor amb menys connexions actives:

```nginx
upstream apache_cluster {
    least_conn;
    server apache1:80;
    server apache2:80;
    keepalive 32;
}
```

**Memòria cau** en disc. Les respostes es guarden 10 minuts. Si arriba una petició per un recurs que ja s'ha servit fa poc, Nginx el retorna directament sense anar als Apaches:

```nginx
proxy_cache_path /var/cache/nginx
    levels=1:2
    keys_zone=mi_cache:10m
    max_size=500m
    inactive=60m
    use_temp_path=off;
```

**Capçaleres personalitzades** per verificar des de fora quin node va respondre i si la resposta ve de la caché:

```nginx
add_header X-Cache-Status  $upstream_cache_status always;
add_header X-Upstream-Node $upstream_addr         always;
add_header X-Served-By     "Nginx-LoadBalancer"   always;
```

---

## Fase 2 — Configuració dels Apaches i volum compartit

### Què hem fet

Els dos Apaches munten exactament la mateixa carpeta `./web` en mode bind. Això significa que serveixen el mateix `index.html`, les mateixes imatges i el mateix vídeo:

```yaml
volumes:
  web_content:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: ./web
```


---

## Contingut de la pàgina i recursos locals

### Què hem fet

La pàgina inclou 3 imatges de paisatges i un vídeo. En un primer moment les imatges venien d'Unsplash i el vídeo era un iframe de YouTube, però això feia que la caché de Nginx no els pogués guardar perquè no els servia ell.

Per solucionar-ho hem descarregat tots els recursos localment a la carpeta `./web`:

```bash
wget -O imagen1.jpg "https://images.unsplash.com/photo-1506905925346-21bda4d32df4?w=900"
wget -O imagen2.jpg "https://images.unsplash.com/photo-1501854140801-50d01698950b?w=700"
wget -O imagen3.jpg "https://images.unsplash.com/photo-1469474968028-56623f02e42e?w=700"
wget -O video.mp4 "https://www.w3schools.com/html/mov_bbb.mp4"
```

I hem modificat el `index.html` perquè apunti als fitxers locals en lloc de les URLs externes. Així Nginx els serveix ell mateix i els pot guardar a la caché.

---

## Desplegament


---

## Evidències de funcionament

### 1. Web funcionant

La pàgina és accessible a `http://localhost:8080`. Es veuen les tres imatges de paisatges, el vídeo i la cita filosòfica.

<img width="1280" height="813" alt="image" src="https://github.com/user-attachments/assets/e18cc83b-64d9-4f88-a6c2-9fe09d7c963b" />

---

### 2. Contenidors en marxa i capçaleres HTTP

Els tres contenidors estan en estat `Running`. El `curl -I` confirma que la petició passa per Nginx (`Server: nginx/1.29.8`, `X-Served-By: Nginx-LoadBalancer`) i que la caché funciona (`X-Cache-Status: HIT`).

![Contenidors i capçaleres](captures/curl-headers.png)

---

### 3. Caché de les imatges — HIT

Les imatges estan servides localment. Després de la primera petició (`MISS`), les següents les retorna Nginx directament des de la caché (`HIT`) sense tocar els Apaches. Es pot veure que `Content-Type: image/jpeg` i `X-Cache-Status: HIT`.

<img width="955" height="801" alt="image" src="https://github.com/user-attachments/assets/dd042395-ff22-4ac7-aac1-6ac6012e857e" />


---

### 4. Caché del vídeo — MISS i HIT

El vídeo també està servit localment (`Content-Type: video/mp4`). La primera petició mostra `X-Cache-Status: MISS` i `X-Upstream-Node: 172.19.0.2:80`, que és la IP del contenidor Apache que va respondre. La segona petició ja serà `HIT`.

<img width="1355" height="660" alt="image" src="https://github.com/user-attachments/assets/f978fb87-356e-4f68-8c5e-aed7a8c04c18" />


---

### 5. Logs — balanceig visible entre apache1 i apache2

Als logs es pot veure com les peticions es reparteixen entre els dos nodes. Un cop la caché expira, `apache1` i `apache2` alternen responent a les mateixes peticions:

<img width="1322" height="732" alt="image" src="https://github.com/user-attachments/assets/b88ab577-e55c-44d6-abeb-4cbf84d2d50d" />
<img width="1310" height="821" alt="image" src="https://github.com/user-attachments/assets/455885f0-4c7c-4625-8506-13c13d3d5b91" />
<img width="1278" height="786" alt="image" src="https://github.com/user-attachments/assets/152fa51a-ccb5-479d-abc4-47dae317d85f" />

