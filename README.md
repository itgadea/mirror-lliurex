# Mirror Local de LliureX — Documentación

Mirror local de los repositorios APT de LliureX (Noble y Jammy) usando Docker,
con sincronización nocturna automática y servicio HTTP mediante Nginx.

---

## Estructura del proyecto

```
mirror-lliurex/
├── docker-compose.yml
├── config/
│   ├── mirror.list       # Configuración de apt-mirror
│   └── nginx.conf        # Configuración del servidor web
├── mirror-data/          # Datos del mirror (puede ocupar varios GB)
│   └── mirror/
│       └── lliurex.net/
│           ├── noble/
│           └── jammy/
└── logs/                 # Logs de sincronización
    ├── sync.log
    └── cron.log
```

Crear la estructura inicial:

```bash
mkdir -p ~/mirror-lliurex/{mirror-data,config,logs}
cd ~/mirror-lliurex
```

---

## Repositorios sincronizados

| Suite | Componentes |
|---|---|
| `noble` | main restricted universe multiverse preschool |
| `noble-updates` | main restricted universe multiverse |
| `noble-security` | main restricted universe multiverse |
| `jammy` | main restricted universe multiverse preschool |
| `jammy-updates` | main restricted universe multiverse |
| `jammy-security` | main restricted universe multiverse |
| `jammy rescue` | — |

> **Nota:** `noble-backports` y `jammy-backports` no existen en el servidor de LliureX
> (devuelven 404), por lo que no se sincronizan.

---

## Archivos de configuración

### `config/mirror.list`

```
set base_path    /var/spool/apt-mirror
set nthreads     20
set _tilde       0

# ── LliureX Noble (25) ──────────────────────────────
deb http://lliurex.net/noble noble main restricted universe multiverse preschool
deb http://lliurex.net/noble noble-updates main restricted universe multiverse
deb http://lliurex.net/noble noble-security main restricted universe multiverse

# ── LliureX Jammy ───────────────────────────────────
deb http://lliurex.net/jammy jammy main restricted universe multiverse preschool
deb http://lliurex.net/jammy jammy-updates main restricted universe multiverse
deb http://lliurex.net/jammy jammy-security main restricted universe multiverse
deb http://lliurex.net/jammy jammy rescue

clean http://lliurex.net/noble
clean http://lliurex.net/jammy
```

### `config/nginx.conf`

```nginx
server {
    listen 80;
    server_name _;

    root /usr/share/nginx/html;
    autoindex on;

    access_log /var/log/nginx/mirror_access.log;
    error_log  /var/log/nginx/mirror_error.log;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

### `docker-compose.yml`

```yaml
services:

  # Sincronizador — se ejecuta y termina
  # Lanzar manualmente o via cron del host
  apt-mirror:
    image: ubuntu:24.04
    container_name: apt-mirror
    volumes:
      - ./mirror-data:/var/spool/apt-mirror
      - ./config/mirror.list:/etc/apt/mirror.list:ro
      - ./logs:/var/log/apt-mirror
    command: >
      bash -c "
        apt-get update -qq &&
        apt-get install -y -qq apt-mirror &&
        apt-mirror 2>&1 | tee /var/log/apt-mirror/sync.log
      "
    restart: "no"
    networks:
      - mirror-net

  # Servidor web — siempre activo
  nginx:
    image: nginx:alpine
    container_name: mirror-nginx
    ports:
      - "8080:80"
    volumes:
      - ./mirror-data/mirror:/usr/share/nginx/html:ro
      - ./config/nginx.conf:/etc/nginx/conf.d/default.conf:ro
    restart: unless-stopped
    networks:
      - mirror-net

networks:
  mirror-net:
    driver: bridge
```

---

## Dry-run — estimación de espacio antes de descargar

Antes de lanzar la primera sincronización real, conviene estimar cuánto va a
ocupar el mirror:

```bash
docker compose run --rm apt-mirror bash -c "
  apt-get update -qq &&
  apt-get install -y -qq apt-mirror &&
  apt-mirror --dry-run
"
```

La salida mostrará el número de paquetes y el tamaño total estimado en GB.
Asegúrate de tener suficiente espacio libre en el disco antes de continuar.

---

## Arranque

### 1. Primera sincronización (hacerla manualmente)

La primera descarga puede tardar varias horas dependiendo del ancho de banda.
Se recomienda lanzarla en una sesión `screen` o `tmux`:

```bash
# Opcional pero recomendable para no perder la sesión
screen -S mirror-sync

# Lanzar la sincronización
docker compose run --rm apt-mirror

# Salir de screen sin matar el proceso: Ctrl+A, D
# Volver a la sesión: screen -r mirror-sync
```

### 2. Levantar el servidor Nginx

```bash
docker compose up -d nginx
```

### 3. Verificar que el mirror es accesible

```bash
curl -I http://localhost:8080/lliurex.net/noble/dists/noble/Release
curl -I http://localhost:8080/lliurex.net/jammy/dists/jammy/Release
```

Ambos deben devolver `HTTP/1.1 200 OK`.

---

## Sincronización automática — Crontab

Editar el crontab del usuario que gestiona el servidor:

```bash
crontab -e
```

Añadir la siguiente línea para sincronizar cada noche a las 2:00 AM:

```
0 2 * * * cd /home/TU_USUARIO/mirror-lliurex && docker compose run --rm apt-mirror >> /home/TU_USUARIO/mirror-lliurex/logs/cron.log 2>&1
```

> Sustituye `TU_USUARIO` por el usuario real del servidor.

Para verificar que el cron está activo:

```bash
crontab -l
```

Los logs de cada sincronización se guardan en:

- `logs/sync.log` — salida completa de apt-mirror
- `logs/cron.log` — salida del cron (errores de arranque del contenedor, etc.)

---

## Configuración de clientes

### Equipos con LliureX Noble (25)

Editar `/etc/apt/sources.list.d/lliurex.sources` y cambiar la URI:

```ini
Types: deb
URIs: http://IP-SERVIDOR:8080/lliurex.net/noble
Suites: noble noble-security noble-updates
Components: main multiverse preschool restricted universe
Enabled: yes
Signed-By: /etc/apt/trusted.gpg.d/lliurex-archive-keyring.gpg
```

### Equipos con LliureX Jammy

Editar `/etc/apt/sources.list` y sustituir las líneas de LliureX:

```
deb http://IP-SERVIDOR:8080/lliurex.net/jammy jammy main restricted universe multiverse preschool
deb http://IP-SERVIDOR:8080/lliurex.net/jammy jammy-updates main restricted universe multiverse
deb http://IP-SERVIDOR:8080/lliurex.net/jammy jammy-security main restricted universe multiverse
deb http://IP-SERVIDOR:8080/lliurex.net/jammy jammy rescue
```

### Verificar en el cliente

```bash
sudo apt update
```

No deben aparecer errores de conexión ni de firma GPG. Si todo va bien, los
equipos descargarán los paquetes desde el mirror local en lugar de Internet.

---

## Integración con Nginx Proxy Manager (opcional)

Si el servidor ya tiene Nginx Proxy Manager, se puede exponer el mirror con un
nombre de dominio interno en lugar del puerto 8080:

1. Crear un nuevo **Proxy Host** en NPM:
   - Domain: `mirror.tucentro.local` (o similar)
   - Forward Hostname: `mirror-nginx`
   - Forward Port: `80`
2. Asegurarse de que el contenedor `mirror-nginx` está en la misma red Docker
   que NPM, o usar la IP del host con el puerto `8080`.
3. En los clientes, usar `http://mirror.tucentro.local` como URI en lugar de
   `http://IP-SERVIDOR:8080`.

---

## Comandos útiles

```bash
# Ver logs de la última sincronización
tail -f logs/sync.log

# Ver estado del contenedor nginx
docker compose ps

# Reiniciar nginx
docker compose restart nginx

# Sincronización manual fuera del cron
docker compose run --rm apt-mirror

# Ver cuánto ocupa el mirror en disco
du -sh mirror-data/
```
