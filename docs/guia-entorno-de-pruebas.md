# Guía de instalación del entorno de pruebas

**Sistema de Captura de Informes — IPACOOP**
Dirigida a: equipo técnico / infraestructura · Versión 1 · 10 de noviembre de 2026
Objetivo: dejar el entorno de **pruebas** listo y verificado para recibir la **primera instalación de R1 (Plan Anual)**. El mismo procedimiento sirve después para **producción**.

---

## 0. Resumen

| Elemento | Definición |
|---|---|
| Servidores | **VM-APP** (aplicación web) y **VM-BD** (base de datos) |
| Sistema operativo | **Ubuntu Server 24.04 LTS** en ambas (si la granja usa RHEL/Rocky 9, ver §11) |
| Base de datos | **PostgreSQL 16** (incluido en Ubuntu 24.04) |
| Aplicación | **Python 3.12** (incluido en Ubuntu 24.04) + **Django 5.2 LTS**, servida por **Gunicorn** |
| Servidor web | **Nginx** (incluido en Ubuntu 24.04) con HTTPS |
| Licencias | Ninguna: todo es software libre |

```
 Usuarios (red institucional)
        │  HTTPS 443
        ▼
 ┌───────────────────────────┐        5432 (solo desde VM-APP)       ┌──────────────────────┐
 │ VM-APP                    │ ─────────────────────────────────────▶ │ VM-BD                │
 │ Nginx → Gunicorn → Django │                                        │ PostgreSQL 16        │
 └───────────────────────────┘                                        │ respaldo diario      │
                                                                      └──────────────────────┘
```

### Calendario

| Fecha | Hito | Responsable |
|---|---|---|
| **Mié 18-nov** | VM-APP y VM-BD entregadas con acceso SSH, red y DNS (§1) | Infraestructura |
| **Vie 20-nov** | Pasos §2 a §8 completos y **verificación del entorno** aprobada (§9) | Equipo técnico |
| **Mar 24-nov** | Primera instalación de R1 en pruebas (§10) | Equipo técnico + desarrollo |
| 25–27-nov | Pruebas de aceptación con Planificación y provincias piloto | Funcional |
| Lun 30-nov | Instalación en producción (mismo procedimiento) | Equipo técnico |
| **Mar 1-dic** | Plan Anual disponible para las direcciones provinciales | — |

### Convenciones de esta guía
- Los valores entre `<…>` se reemplazan por los datos reales (ver la hoja de datos en §12).
- `ENTORNO` vale `pruebas` en esta guía y `produccion` cuando se repita para producción.
- Los comandos se ejecutan como un usuario con `sudo`.

---

## 1. Requisitos previos (infraestructura)

### 1.1 Máquinas virtuales
| | VM-APP | VM-BD |
|---|---|---|
| vCPU | 2 | 2 |
| RAM | 4 GB | 4–8 GB |
| Disco | 40 GB | 60 GB (+ espacio de respaldos, §5) |
| SO | Ubuntu Server 24.04 LTS, instalación mínima, con OpenSSH | Igual |

### 1.2 Red y accesos
| Origen → Destino | Puerto | Uso |
|---|---|---|
| Usuarios (red institucional) → VM-APP | 443 (y 80 solo para redirigir a 443) | Acceso al sistema |
| VM-APP → VM-BD | 5432 | Conexión de la aplicación a la base |
| Administradores → ambas | 22 | Administración por SSH |
| Ambas → internet (o espejo interno) | 443/80 | Instalar paquetes `apt` y `pip`; descargar el código (o copiarlo, ver §10.1) |

### 1.3 Otros
- **Nombre DNS interno** para pruebas, p. ej. `<FQDN_PRUEBAS>` = `captura-pruebas.ipacoop.gob.pa` (y otro para producción).
- **Certificado TLS** para ese nombre emitido por la CA institucional (archivo `.crt` con la cadena y archivo `.key`). Si aún no existe, §7.3 explica cómo usar uno temporal autofirmado.
- **Hora sincronizada (NTP)** y zona horaria `America/Panama`.
- Opcional: servidor **SMTP** institucional (para avisos por correo en R4) y **almacenamiento externo** (NAS/carpeta compartida) para copiar los respaldos.

---

## 2. Configuración base (en AMBAS VM)

```bash
# 2.1 Actualizar el sistema
sudo apt update && sudo apt -y full-upgrade

# 2.2 Zona horaria, idioma y hora
sudo timedatectl set-timezone America/Panama
sudo locale-gen es_PA.UTF-8 en_US.UTF-8
sudo update-locale LANG=en_US.UTF-8
timedatectl            # verificar "System clock synchronized: yes"

# 2.3 Actualizaciones de seguridad automáticas
sudo apt -y install unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades

# 2.4 Utilidades
sudo apt -y install curl ca-certificates git rsync htop

# 2.5 Nombre del equipo (uno en cada VM)
sudo hostnamectl set-hostname <vm-app-pruebas | vm-bd-pruebas>
```

**SSH:** acceso solo con llave; deshabilitar el ingreso de `root` y por contraseña:
```bash
sudo sed -i 's/^#\?PermitRootLogin.*/PermitRootLogin no/; s/^#\?PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config
sudo systemctl restart ssh
```
> Antes de aplicar lo anterior, confirme que puede entrar con su llave en otra terminal.

---

## 3. VM-BD — PostgreSQL 16

### 3.1 Instalar
```bash
sudo apt -y install postgresql postgresql-contrib
psql --version                      # debe mostrar 16.x
sudo systemctl enable --now postgresql
```

### 3.2 Aceptar conexiones solo desde VM-APP
```bash
# Escuchar en la IP de la VM-BD
sudo sed -i "s/^#\?listen_addresses.*/listen_addresses = 'localhost,<IP_VM_BD>'/" /etc/postgresql/16/main/postgresql.conf

# Permitir solo a la VM-APP, con contraseña cifrada (scram-sha-256)
echo "host  coop_pruebas  coop_app  <IP_VM_APP>/32  scram-sha-256" | sudo tee -a /etc/postgresql/16/main/pg_hba.conf

sudo systemctl restart postgresql
```

### 3.3 Ajustes básicos de memoria (VM de 4 GB)
```bash
sudo tee /etc/postgresql/16/main/conf.d/coop.conf >/dev/null <<'EOF'
shared_buffers = 1GB
effective_cache_size = 3GB
work_mem = 16MB
maintenance_work_mem = 256MB
timezone = 'America/Panama'
log_min_duration_statement = 1000   # registra consultas de más de 1 s
EOF
sudo systemctl restart postgresql
```
(Con 8 GB de RAM: `shared_buffers = 2GB`, `effective_cache_size = 6GB`.)

### 3.4 Crear usuario y base de datos
Genere una contraseña larga y guárdela en el gestor de secretos del área (se usará en §6.3):
```bash
openssl rand -base64 32
```
```bash
sudo -u postgres psql <<'EOF'
CREATE ROLE coop_app LOGIN PASSWORD '<CLAVE_BD>';
CREATE DATABASE coop_pruebas OWNER coop_app ENCODING 'UTF8' TEMPLATE template0;
\c coop_pruebas
CREATE EXTENSION IF NOT EXISTS unaccent;   -- búsquedas sin acentos
CREATE EXTENSION IF NOT EXISTS pg_trgm;    -- búsqueda aproximada de cooperativas
EOF
```
> `coop_app` no es superusuario: solo es dueño de su base. Las extensiones las crea el administrador aquí, una sola vez.

### 3.5 Cortafuegos
```bash
sudo ufw allow OpenSSH
sudo ufw allow from <IP_VM_APP> to any port 5432 proto tcp
sudo ufw enable
sudo ufw status
```

---

## 4. VM-APP — paquetes del sistema

```bash
sudo apt -y install python3 python3-venv python3-dev build-essential libpq-dev \
                    postgresql-client nginx
python3 --version     # debe mostrar 3.12.x
nginx -v
```

Cortafuegos:
```bash
sudo ufw allow OpenSSH
sudo ufw allow 'Nginx Full'      # 80 y 443
sudo ufw enable
```

Probar la conexión a la base desde la VM-APP:
```bash
psql "postgresql://coop_app@<IP_VM_BD>:5432/coop_pruebas" -c "select version();"
# pedirá <CLAVE_BD>; debe responder con la versión de PostgreSQL 16
```

---

## 5. VM-BD — respaldo diario

```bash
sudo mkdir -p /var/backups/coop && sudo chown postgres:postgres /var/backups/coop

sudo tee /usr/local/bin/coop-respaldo.sh >/dev/null <<'EOF'
#!/bin/bash
# Respaldo diario de la base del Sistema de Captura de Informes
set -euo pipefail
BASE="${1:-coop_pruebas}"
DESTINO=/var/backups/coop
ARCHIVO="$DESTINO/${BASE}_$(date +%Y%m%d_%H%M).dump"
pg_dump -Fc -d "$BASE" -f "$ARCHIVO"
# Conservar 30 días
find "$DESTINO" -name "${BASE}_*.dump" -mtime +30 -delete
# Copia fuera del servidor (descomentar y ajustar cuando exista el destino)
# rsync -a "$ARCHIVO" <USUARIO>@<SERVIDOR_RESPALDO>:/respaldos/coop/
EOF
sudo chmod 755 /usr/local/bin/coop-respaldo.sh

# Programar todos los días a la 1:30 a.m.
echo "30 1 * * * postgres /usr/local/bin/coop-respaldo.sh coop_pruebas" | sudo tee /etc/cron.d/coop-respaldo

# Probar ahora
sudo -u postgres /usr/local/bin/coop-respaldo.sh coop_pruebas && ls -lh /var/backups/coop
```

**Prueba de restauración** (obligatoria antes de R1; se restaura en una base temporal):
```bash
sudo -u postgres createdb coop_restauracion
sudo -u postgres pg_restore -d coop_restauracion --no-owner /var/backups/coop/<ARCHIVO>.dump
sudo -u postgres psql -d coop_restauracion -c "\dt"
sudo -u postgres dropdb coop_restauracion
```

---

## 6. VM-APP — estructura de la aplicación

### 6.1 Usuario de servicio y carpetas
```bash
sudo adduser --system --group --home /srv/coop --shell /usr/sbin/nologin coop
sudo mkdir -p /srv/coop/pruebas/{releases,shared/static,shared/media,shared/logs}
sudo chown -R coop:coop /srv/coop
sudo mkdir -p /etc/coop && sudo chmod 750 /etc/coop && sudo chown root:coop /etc/coop
```

Estructura resultante:
```
/srv/coop/pruebas/
├── releases/            # una carpeta por versión instalada (p. ej. 2026-11-24_r1)
├── current -> releases/<versión activa>   # enlace simbólico a la versión en uso
└── shared/
    ├── static/          # archivos estáticos (los sirve Nginx)
    ├── media/           # adjuntos (anexos de informes técnicos)
    └── logs/            # registros de la aplicación
```

### 6.2 Clave secreta de Django
```bash
python3 -c "import secrets; print(secrets.token_urlsafe(50))"
```

### 6.3 Archivo de configuración del entorno
La aplicación lee **únicamente** estas variables (contrato acordado con el equipo de desarrollo):

```bash
sudo tee /etc/coop/pruebas.env >/dev/null <<'EOF'
COOP_ENTORNO=pruebas
DJANGO_SETTINGS_MODULE=config.settings
DJANGO_SECRET_KEY=<CLAVE_SECRETA_DJANGO>
DJANGO_DEBUG=0
DJANGO_ALLOWED_HOSTS=<FQDN_PRUEBAS>
DJANGO_CSRF_TRUSTED_ORIGINS=https://<FQDN_PRUEBAS>
DATABASE_URL=postgres://coop_app:<CLAVE_BD>@<IP_VM_BD>:5432/coop_pruebas
COOP_STATIC_ROOT=/srv/coop/pruebas/shared/static
COOP_MEDIA_ROOT=/srv/coop/pruebas/shared/media
COOP_LOG_DIR=/srv/coop/pruebas/shared/logs
# Opcional (R4): avisos por correo
# EMAIL_HOST=<SMTP>
# EMAIL_PORT=587
# EMAIL_HOST_USER=
# EMAIL_HOST_PASSWORD=
# DEFAULT_FROM_EMAIL=captura@ipacoop.gob.pa
EOF
sudo chown root:coop /etc/coop/pruebas.env
sudo chmod 640 /etc/coop/pruebas.env
```
> Si `<CLAVE_BD>` contiene caracteres especiales (`@ : / # ?`), codifíquelos en formato URL o genere una clave solo con letras y números.

---

## 7. VM-APP — Gunicorn y Nginx

### 7.1 Servicio Gunicorn (systemd)
```bash
sudo tee /etc/systemd/system/coop-pruebas.service >/dev/null <<'EOF'
[Unit]
Description=Sistema de Captura de Informes IPACOOP (pruebas)
After=network.target

[Service]
Type=notify
User=coop
Group=coop
RuntimeDirectory=coop-pruebas
WorkingDirectory=/srv/coop/pruebas/current
EnvironmentFile=/etc/coop/pruebas.env
ExecStart=/srv/coop/pruebas/current/.venv/bin/gunicorn config.wsgi:application \
          --bind unix:/run/coop-pruebas/gunicorn.sock \
          --workers 3 --timeout 60 \
          --access-logfile /srv/coop/pruebas/shared/logs/gunicorn-acceso.log \
          --error-logfile  /srv/coop/pruebas/shared/logs/gunicorn-error.log
ExecReload=/bin/kill -s HUP $MAINPID
NotifyAccess=main
Restart=on-failure
PrivateTmp=true
NoNewPrivileges=true

[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
```
(Se habilita en §9 o §10, cuando exista código en `current`.)

### 7.2 Certificado TLS
Copiar el certificado de la CA institucional:
```bash
sudo mkdir -p /etc/ssl/coop
sudo cp <certificado_con_cadena>.crt /etc/ssl/coop/pruebas.crt
sudo cp <llave_privada>.key          /etc/ssl/coop/pruebas.key
sudo chmod 600 /etc/ssl/coop/pruebas.key
```

### 7.3 (Solo si aún no hay certificado) certificado temporal autofirmado
```bash
sudo openssl req -x509 -nodes -newkey rsa:2048 -days 90 \
  -keyout /etc/ssl/coop/pruebas.key -out /etc/ssl/coop/pruebas.crt \
  -subj "/CN=<FQDN_PRUEBAS>"
```
> El navegador mostrará una advertencia. Reemplazar por el certificado institucional antes de las pruebas de aceptación.

### 7.4 Sitio de Nginx
```bash
sudo tee /etc/nginx/sites-available/coop-pruebas >/dev/null <<'EOF'
server {
    listen 80;
    server_name <FQDN_PRUEBAS>;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name <FQDN_PRUEBAS>;

    ssl_certificate     /etc/ssl/coop/pruebas.crt;
    ssl_certificate_key /etc/ssl/coop/pruebas.key;
    ssl_protocols TLSv1.2 TLSv1.3;

    add_header Strict-Transport-Security "max-age=31536000" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-Frame-Options "DENY" always;
    add_header Referrer-Policy "same-origin" always;

    client_max_body_size 12m;          # anexos de hasta 10 MB
    access_log /var/log/nginx/coop-pruebas-acceso.log;
    error_log  /var/log/nginx/coop-pruebas-error.log;

    location /static/ {
        alias /srv/coop/pruebas/shared/static/;
        expires 7d;
    }

    # Los adjuntos NO se publican directamente: la aplicación verifica permisos
    # y Nginx los entrega mediante X-Accel-Redirect.
    location /protegido/ {
        internal;
        alias /srv/coop/pruebas/shared/media/;
    }

    location / {
        proxy_pass http://unix:/run/coop-pruebas/gunicorn.sock;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
        proxy_read_timeout 60s;
    }
}
EOF
sudo ln -s /etc/nginx/sites-available/coop-pruebas /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t && sudo systemctl reload nginx
```

---

## 8. Registros y espacio en disco

```bash
# Rotación de los registros de la aplicación
sudo tee /etc/logrotate.d/coop >/dev/null <<'EOF'
/srv/coop/*/shared/logs/*.log {
    weekly
    rotate 12
    compress
    missingok
    notifempty
    copytruncate
}
EOF
```
Consultas útiles:
```bash
sudo journalctl -u coop-pruebas -f          # servicio de la aplicación
sudo tail -f /var/log/nginx/coop-pruebas-error.log
df -h                                        # espacio en disco (alertar al 80 %)
```

---

## 9. Verificación del entorno (antes de R1)

Como el código de R1 aún no existe, se instala un **proyecto Django mínimo de verificación** que recorre toda la cadena (Nginx → Gunicorn → Django → PostgreSQL). Al terminar se borra.

```bash
# 9.1 Crear la versión de verificación
sudo -u coop bash <<'EOF'
set -e
cd /srv/coop/pruebas/releases
mkdir verificacion && cd verificacion
python3 -m venv .venv
.venv/bin/pip install --upgrade pip
.venv/bin/pip install "django>=5.2,<5.3" gunicorn "psycopg[binary]" dj-database-url
.venv/bin/django-admin startproject config .
cat >> config/settings.py <<'PY'

# --- Configuración por variables de entorno (verificación) ---
import os, dj_database_url
SECRET_KEY = os.environ["DJANGO_SECRET_KEY"]
DEBUG = os.environ.get("DJANGO_DEBUG", "0") == "1"
ALLOWED_HOSTS = os.environ["DJANGO_ALLOWED_HOSTS"].split(",")
CSRF_TRUSTED_ORIGINS = os.environ["DJANGO_CSRF_TRUSTED_ORIGINS"].split(",")
DATABASES = {"default": dj_database_url.parse(os.environ["DATABASE_URL"])}
STATIC_ROOT = os.environ["COOP_STATIC_ROOT"]
SECURE_PROXY_SSL_HEADER = ("HTTP_X_FORWARDED_PROTO", "https")
SESSION_COOKIE_SECURE = CSRF_COOKIE_SECURE = True
TIME_ZONE = "America/Panama"
LANGUAGE_CODE = "es"
PY
EOF
sudo -u coop ln -sfn /srv/coop/pruebas/releases/verificacion /srv/coop/pruebas/current

# 9.2 Migrar, recolectar estáticos y crear un usuario administrador de prueba
cd /srv/coop/pruebas/current
sudo -u coop bash -c 'set -a; . /etc/coop/pruebas.env; set +a;
  .venv/bin/python manage.py migrate &&
  .venv/bin/python manage.py collectstatic --noinput &&
  .venv/bin/python manage.py check --deploy'
sudo -u coop bash -c 'set -a; . /etc/coop/pruebas.env; set +a; .venv/bin/python manage.py createsuperuser'

# 9.3 Levantar el servicio
sudo systemctl enable --now coop-pruebas
sudo systemctl status coop-pruebas --no-pager
```

### 9.4 Lista de comprobación
| # | Prueba | Resultado esperado | OK |
|---|---|---|---|
| 1 | `curl -I http://<FQDN_PRUEBAS>` | `301` hacia `https://` | ☐ |
| 2 | `curl -kI https://<FQDN_PRUEBAS>/admin/` | `302` hacia la página de ingreso | ☐ |
| 3 | Abrir `https://<FQDN_PRUEBAS>/admin/` desde un PC de una provincia | Página de ingreso de Django con estilos | ☐ |
| 4 | Ingresar con el usuario creado en 9.2 | Entra al panel de administración (prueba la base de datos) | ☐ |
| 5 | `sudo systemctl restart coop-pruebas` y recargar | El sitio vuelve en segundos | ☐ |
| 6 | Reiniciar la VM-APP completa | El sitio vuelve solo | ☐ |
| 7 | Desde un equipo distinto de VM-APP: `nc -zv <IP_VM_BD> 5432` | **Rechazado** (solo VM-APP accede a la base) | ☐ |
| 8 | Respaldo de la noche anterior presente en `/var/backups/coop` | Archivo `.dump` del día | ☐ |
| 9 | Prueba de restauración de §5 | Tablas listadas correctamente | ☐ |
| 10 | Certificado institucional instalado | El navegador no muestra advertencia | ☐ |

`check --deploy` (9.2) puede mostrar avisos sobre `SECURE_HSTS_SECONDS` y `SECURE_SSL_REDIRECT`: son esperados en la verificación (HSTS y la redirección los hace Nginx); la aplicación real los configurará.

### 9.5 Limpieza después de aprobar
En **VM-APP**, detener el servicio:
```bash
sudo systemctl stop coop-pruebas
```
Para dejar la base vacía para R1, en **VM-BD**:
```bash
sudo -u postgres psql <<'EOF'
DROP DATABASE coop_pruebas;
CREATE DATABASE coop_pruebas OWNER coop_app ENCODING 'UTF8' TEMPLATE template0;
\c coop_pruebas
CREATE EXTENSION IF NOT EXISTS unaccent;
CREATE EXTENSION IF NOT EXISTS pg_trgm;
EOF
```
Y en **VM-APP**:
```bash
sudo rm -rf /srv/coop/pruebas/releases/verificacion /srv/coop/pruebas/shared/static/*
sudo rm -f /srv/coop/pruebas/current
```

---

## 10. Instalación de una versión (R1 y siguientes)

El equipo de desarrollo entregará cada versión con una **etiqueta** en el repositorio (p. ej. `r1.0.0`) y sus notas.

### 10.1 Obtener el código (una de dos opciones)
- **A. Desde GitHub** (si la VM-APP tiene salida a internet): crear una *deploy key* de solo lectura para el repositorio `easprillacoop/COOP_TEST` con la llave pública de `/srv/coop/.ssh/id_ed25519.pub`.
- **B. Sin internet:** descargar el `.zip` de la etiqueta en un equipo con acceso y copiarlo con `scp` a `/tmp` de la VM-APP.

### 10.2 Procedimiento
```bash
VERSION=r1.0.0
DESTINO=/srv/coop/pruebas/releases/$(date +%Y-%m-%d)_$VERSION

# Opción A
sudo -u coop git clone --depth 1 --branch $VERSION git@github.com:easprillacoop/COOP_TEST.git $DESTINO
# Opción B
# sudo -u coop mkdir -p $DESTINO && sudo -u coop unzip /tmp/COOP_TEST-$VERSION.zip -d $DESTINO

cd $DESTINO
sudo -u coop python3 -m venv .venv
sudo -u coop .venv/bin/pip install --upgrade pip
sudo -u coop .venv/bin/pip install -r requirements.txt

# Migraciones, estáticos y verificación
sudo -u coop bash -c 'set -a; . /etc/coop/pruebas.env; set +a;
  .venv/bin/python manage.py migrate &&
  .venv/bin/python manage.py collectstatic --noinput &&
  .venv/bin/python manage.py check --deploy'

# Activar la nueva versión (corte de pocos segundos)
sudo -u coop ln -sfn $DESTINO /srv/coop/pruebas/current
sudo systemctl restart coop-pruebas
curl -k https://<FQDN_PRUEBAS>/salud/        # debe responder: ok
```

### 10.3 Solo en la primera instalación (R1)
```bash
cd /srv/coop/pruebas/current
sudo -u coop bash -c 'set -a; . /etc/coop/pruebas.env; set +a;
  .venv/bin/python manage.py cargar_catalogos &&       # catálogos y cooperativas (referencia/seeds)
  .venv/bin/python manage.py createsuperuser'          # cuenta del Administrador TI
```
Luego, el Administrador TI entra a `https://<FQDN_PRUEBAS>/admin/` y crea las cuentas de la lista de usuarios (o se cargan con el comando que se entregue en R1).

### 10.4 Volver a la versión anterior (si algo falla)
```bash
ls /srv/coop/pruebas/releases                       # identificar la versión previa
sudo -u coop ln -sfn /srv/coop/pruebas/releases/<VERSION_ANTERIOR> /srv/coop/pruebas/current
sudo systemctl restart coop-pruebas
```
> Si la versión nueva incluía migraciones de base de datos, restaurar además el respaldo tomado justo antes de instalar. Por eso, **antes de cada instalación** se ejecuta en VM-BD: `sudo -u postgres /usr/local/bin/coop-respaldo.sh coop_pruebas`.

---

## 11. Si la granja usa RHEL / Rocky / Alma Linux 9

| Paso | Ubuntu 24.04 | RHEL/Rocky 9 |
|---|---|---|
| Paquetes | `apt install …` | `dnf install …` |
| PostgreSQL 16 | `postgresql` | `dnf module enable postgresql:16 && dnf install postgresql-server postgresql-contrib && postgresql-setup --initdb` |
| Rutas PostgreSQL | `/etc/postgresql/16/main/` | `/var/lib/pgsql/data/` |
| Python 3.12 | `python3` | `dnf install python3.12 python3.12-devel` y usar `python3.12` |
| Cortafuegos | `ufw` | `firewall-cmd` |
| Nginx | `sites-available` | `/etc/nginx/conf.d/coop-pruebas.conf` |
| SELinux | — | `setsebool -P httpd_can_network_connect 1` y permitir el socket de Gunicorn |

El resto del procedimiento es igual.

---

## 12. Hoja de datos a devolver al equipo de desarrollo

| Dato | Valor |
|---|---|
| Sistema operativo y versión de cada VM | |
| IP de VM-APP / IP de VM-BD | |
| `<FQDN_PRUEBAS>` y `<FQDN_PRODUCCION>` | |
| ¿VM-APP tiene salida a internet? (opción A o B de §10.1) | |
| Certificado institucional instalado (sí/no) | |
| Servidor SMTP disponible (sí/no, host y puerto) | |
| Destino externo de respaldos | |
| Persona responsable técnica y medio de contacto | |
| Fecha de aprobación de la verificación (§9.4) | |

> **No enviar contraseñas ni claves por correo.** Las claves (`<CLAVE_BD>`, `<CLAVE_SECRETA_DJANGO>`) se quedan en el servidor y en el gestor de secretos del área.

---

## 13. Producción

Repetir esta guía reemplazando `pruebas` por `produccion` en nombres de base de datos, carpetas, archivo `.env`, servicio, sitio de Nginx, certificado y respaldo. Se recomienda un par de VM propio para producción; si se comparte el mismo par con pruebas, los dos entornos quedan aislados por base de datos, usuario de sistema, carpeta, servicio y nombre DNS.

---

### Anexo — Compromisos del equipo de desarrollo con este entorno

Para que la primera instalación funcione sin ajustes, la aplicación R1 se construirá cumpliendo:
1. Configuración **solo** por las variables de §6.3; ningún secreto en el código.
2. Módulo WSGI `config.wsgi:application` y `requirements.txt` con versiones fijas.
3. Comando `manage.py cargar_catalogos` para la carga inicial y `manage.py check --deploy` sin errores.
4. Ruta `/salud/` que responde `ok` cuando la aplicación y la base de datos funcionan.
5. Adjuntos servidos mediante `X-Accel-Redirect` a `/protegido/`, tras validar permisos.
6. Notas de versión con cada etiqueta, indicando si incluye migraciones de base de datos.
