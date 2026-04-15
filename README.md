# Ansible-Proyecto

Colección de roles Ansible **reutilizables y cross-platform** (Linux + Windows) para administración de servidores e infraestructura.

---

## Índice

- [Requisitos](#requisitos)
- [Estructura del proyecto](#estructura-del-proyecto)
- [Patrón cross-platform](#patrón-cross-platform)
- [Roles por categoría](#roles-por-categoría)
  - [Sistema](#sistema)
  - [Acceso remoto](#acceso-remoto)
  - [Servidores web](#servidores-web)
  - [Bases de datos](#bases-de-datos)
  - [Docker](#docker)
  - [Seguridad](#seguridad)
  - [Usuarios](#usuarios)
  - [Backup](#backup)
- [Playbooks](#playbooks)
- [Uso rápido](#uso-rápido)
- [Dónde configurar las variables](#dónde-configurar-las-variables)
- [Convenciones](#convenciones)

### Leyenda de plataformas

| Distintivo | Significado |
|---|---|
| `Linux \| Windows` | El rol tiene lógica separada por SO (`linux.yml` + `windows.yml`) |
| `Cross-platform` | El rol usa módulos que funcionan igual en ambos SO, sin split |

---

## Requisitos

- Ansible >= 2.12
- Python >= 3.8

### Colecciones necesarias

```bash
ansible-galaxy collection install \
  ansible.posix \
  community.general \
  community.docker \
  community.postgresql \
  community.mysql \
  community.windows \
  ansible.windows \
  chocolatey.chocolatey
```

---

## Estructura del proyecto

```
Ansible-Proyecto/
├── ansible.cfg
├── inventory/
│   ├── hosts
│   └── group_vars/
│       └── all.yml
├── playbooks/
│   ├── setup_common.yml
│   ├── update_system.yml
│   ├── setup_ssh.yml
│   ├── setup_nginx.yml
│   ├── setup_apache.yml
│   ├── setup_postgresql.yml
│   ├── setup_mariadb.yml
│   ├── setup_docker.yml
│   ├── setup_firewall.yml
│   ├── setup_fail2ban.yml
│   ├── setup_users.yml
│   ├── setup_user_dirs.yml
│   ├── setup_backup_local.yml
│   └── setup_backup_remote.yml
└── roles/
    ├── common/           [Linux | Windows]
    ├── update/           [Linux | Windows]
    ├── ssh/              [Linux | Windows]
    ├── nginx/            [Linux | Windows]
    ├── apache/           [Linux | Windows]
    ├── postgresql/       [Linux | Windows]
    ├── mariadb/          [Linux | Windows]
    ├── docker_install/   [Linux | Windows]
    ├── docker_daemon/    [Linux | Windows]
    ├── docker_network/   [Cross-platform]
    ├── docker_volume/    [Cross-platform]
    ├── docker_container/ [Cross-platform]
    ├── firewall/         [Linux | Windows]
    ├── fail2ban/         [Linux | Windows]
    ├── user_discovery/   [Linux | Windows]
    ├── user_dirs/        [Linux | Windows]
    ├── user_create/      [Linux | Windows]
    ├── user_sudo/        [Linux | Windows]
    ├── backup_local/     [Linux | Windows]
    ├── backup_remote/    [Linux | Windows]
    └── healthcheck/      [Linux | Windows]
```

Estructura interna de cada rol con split de SO:

```
roles/<rol>/
├── tasks/
│   ├── main.yml        ← dispatcher: detecta el SO y delega
│   ├── linux.yml       ← lógica específica Linux
│   └── windows.yml     ← lógica específica Windows
├── defaults/
│   └── main.yml        ← variables (con secciones # ── Linux y # ── Windows)
└── handlers/
    └── main.yml        ← handlers con condición when: ansible_os_family
```

---

## Patrón cross-platform

Todos los roles detectan automáticamente el SO del host y ejecutan las tareas correspondientes. No hay que indicar nada manualmente.

**`tasks/main.yml`** de cada rol:

```yaml
- name: Detectar SO e incluir tareas correspondientes
  ansible.builtin.include_tasks: "{{ 'windows' if ansible_os_family == 'Windows' else 'linux' }}.yml"
```

Los roles `docker_network`, `docker_volume` y `docker_container` no necesitan split porque usan la API de Docker a través de `community.docker`, que funciona igual en ambos sistemas operativos.

---

## Roles por categoría

---

### Sistema

#### `common` `Linux | Windows`
Instala paquetes base y configura zona horaria y NTP.

| Variable | Defecto | Descripción |
|---|---|---|
| `common_packages` | curl, git, vim... | Paquetes a instalar (Linux) |
| `common_packages_windows` | git, curl, 7zip... | Paquetes vía Chocolatey (Windows) |
| `timezone` | `Europe/Madrid` | Zona horaria Linux |
| `timezone_windows` | `Romance Standard Time` | Zona horaria Windows |

---

#### `update` `Linux | Windows`
Actualiza todos los paquetes del sistema.

| Variable | Defecto | Descripción |
|---|---|---|
| `update_upgrade_type` | `dist` | Tipo de upgrade en apt: `safe`, `full`, `dist` |
| `update_autoremove` | `true` | Eliminar paquetes huérfanos |
| `update_allow_reboot` | `false` | Reiniciar automáticamente si es necesario |

---

### Acceso remoto

#### `ssh` `Linux | Windows`
Instala OpenSSH Server y aplica hardening de configuración.

| Variable | Defecto | Descripción |
|---|---|---|
| `ssh_port` | `22` | Puerto de escucha |
| `ssh_permit_root_login` | `no` | Permitir login de root |
| `ssh_password_authentication` | `no` | Autenticación por contraseña |
| `ssh_pubkey_authentication` | `yes` | Autenticación por clave pública |
| `ssh_max_auth_tries` | `3` | Intentos máximos de autenticación |
| `ssh_users` | `[]` | Lista de `{name, public_key}` para autorizar |

```yaml
ssh_users:
  - name: deployer
    public_key: "ssh-rsa AAAA..."
```

---

### Servidores web

#### `nginx` `Linux | Windows`
Instala y configura Nginx con soporte de vhosts.

| Variable | Defecto | Descripción |
|---|---|---|
| `nginx_server_name` | `default` | Nombre del servidor / vhost |
| `nginx_server_port` | `80` | Puerto de escucha |
| `nginx_webroot` | `/var/www/html` | Directorio raíz (Linux) |
| `nginx_webroot_windows` | `C:\tools\nginx\html` | Directorio raíz (Windows) |
| `nginx_deploy_test_page` | `true` | Desplegar página de prueba |

---

#### `apache` `Linux | Windows`
Instala y configura Apache con soporte de VirtualHosts.

| Variable | Defecto | Descripción |
|---|---|---|
| `apache_server_name` | `mi-sitio.local` | Nombre del VirtualHost |
| `apache_server_port` | `80` | Puerto de escucha |
| `apache_webroot` | `/var/www/html` | Directorio raíz (Linux) |
| `apache_webroot_windows` | `C:\Apache24\htdocs` | Directorio raíz (Windows) |
| `apache_modules` | rewrite, headers, ssl | Módulos a habilitar |

---

### Bases de datos

#### `postgresql` `Linux | Windows`
Instala PostgreSQL y gestiona bases de datos y usuarios. Requiere `community.postgresql`.

| Variable | Defecto | Descripción |
|---|---|---|
| `postgresql_version` | `16` | Versión a instalar |
| `postgresql_port` | `5432` | Puerto de escucha |
| `postgresql_listen_addresses` | `localhost` | Interfaces de escucha |
| `postgresql_max_connections` | `100` | Conexiones máximas |
| `postgresql_databases` | `[]` | Bases de datos a crear |
| `postgresql_users` | `[]` | Usuarios a crear |

```yaml
postgresql_databases:
  - name: miapp
    owner: appuser

postgresql_users:
  - name: appuser
    password: "{{ vault_pg_password }}"
    role_attr_flags: NOSUPERUSER,NOCREATEDB
    db: miapp
```

---

#### `mariadb` `Linux | Windows`
Instala MariaDB, aplica securización inicial y gestiona bases de datos y usuarios. Requiere `community.mysql`.

| Variable | Defecto | Descripción |
|---|---|---|
| `mariadb_root_password` | `""` | Contraseña de root (usar Vault) |
| `mariadb_port` | `3306` | Puerto de escucha |
| `mariadb_bind_address` | `127.0.0.1` | Interfaz de escucha |
| `mariadb_databases` | `[]` | Bases de datos a crear |
| `mariadb_users` | `[]` | Usuarios a crear |

```yaml
mariadb_root_password: "{{ vault_mysql_root }}"

mariadb_databases:
  - name: miapp
    encoding: utf8mb4

mariadb_users:
  - name: appuser
    password: "{{ vault_db_password }}"
    host: localhost
    priv: "miapp.*:ALL"
```

---

### Docker

Los roles Docker están pensados para usarse en cadena: primero instalar, luego configurar el daemon y finalmente gestionar redes, volúmenes y contenedores de forma independiente.

#### `docker_install` `Linux | Windows`
Instala Docker Engine (Linux) o Docker Desktop (Windows) y arranca el servicio.

| Variable | Defecto | Descripción |
|---|---|---|
| `docker_packages` | docker-ce, cli, compose... | Paquetes a instalar (Linux) |
| `docker_users` | `[]` | Usuarios añadidos al grupo docker |

---

#### `docker_daemon` `Linux | Windows`
Configura el daemon de Docker mediante `daemon.json`.

| Variable | Defecto | Descripción |
|---|---|---|
| `docker_daemon_options` | log-driver, storage-driver... | Dict completo para `daemon.json` |

```yaml
docker_daemon_options:
  log-driver: json-file
  log-opts:
    max-size: "100m"
    max-file: "3"
  storage-driver: overlay2
  insecure-registries:
    - mi-registry.local:5000
```

---

#### `docker_network` `Cross-platform`
Crea y gestiona redes Docker personalizadas.

```yaml
docker_networks:
  - name: frontend
    driver: bridge
  - name: backend
    driver: bridge
    internal: true
  - name: red-vieja
    state: absent
```

---

#### `docker_volume` `Cross-platform`
Crea y gestiona volúmenes Docker persistentes.

```yaml
docker_volumes:
  - name: datos-postgres
  - name: logs
    driver: local
    driver_options:
      type: none
      device: /srv/logs
      o: bind
```

---

#### `docker_container` `Cross-platform`
Despliega y gestiona contenedores Docker.

```yaml
docker_containers:
  - name: web
    image: nginx:latest
    restart_policy: always
    ports:
      - "80:80"
    volumes:
      - /srv/html:/usr/share/nginx/html:ro
    networks:
      - name: frontend
    memory: "256m"

  - name: db
    image: postgres:16
    restart_policy: unless-stopped
    volumes:
      - datos-postgres:/var/lib/postgresql/data
    env:
      POSTGRES_DB: miapp
      POSTGRES_USER: appuser
      POSTGRES_PASSWORD: "{{ vault_db_password }}"
    networks:
      - name: backend
    memory: "512m"

  - name: contenedor-viejo
    image: viejo:1.0
    state: absent
```

| Clave | Defecto | Descripción |
|---|---|---|
| `state` | `started` | `started` / `stopped` / `absent` |
| `restart_policy` | `unless-stopped` | `always` / `unless-stopped` / `on-failure` / `no` |
| `pull` | `true` | Forzar pull de imagen al arrancar |

---

### Seguridad

#### `firewall` `Linux | Windows`
Gestiona reglas de firewall. Usa `ufw` en Debian/Ubuntu, `firewalld` en RedHat/CentOS y Windows Firewall en Windows.

| Variable | Defecto | Descripción |
|---|---|---|
| `firewall_rules` | `[]` | Lista de reglas a aplicar |
| `firewall_default_input` | `deny` | Política por defecto de entrada (Linux) |
| `firewall_windows_profiles` | Domain, Private, Public | Perfiles a habilitar (Windows) |

```yaml
firewall_rules:
  - port: 22
    proto: tcp
    action: allow
    comment: SSH
  - port: 80
    proto: tcp
    action: allow
    comment: HTTP
  - port: 443
    proto: tcp
    action: allow
    comment: HTTPS
```

---

#### `fail2ban` `Linux | Windows`
Protección contra ataques de fuerza bruta. Usa `fail2ban` en Linux e `IPBan` en Windows.

| Variable | Defecto | Descripción |
|---|---|---|
| `fail2ban_bantime` | `3600` | Segundos de baneo |
| `fail2ban_maxretry` | `5` | Intentos antes del baneo |
| `fail2ban_findtime` | `600` | Ventana de tiempo para contar intentos |
| `fail2ban_jails` | `[sshd]` | Jails a habilitar (Linux) |
| `ipban_bantime` | `3600` | Segundos de baneo (Windows) |
| `ipban_maxretry` | `5` | Intentos antes del baneo (Windows) |

```yaml
fail2ban_jails:
  - name: sshd
    enabled: true
    port: ssh
    logpath: /var/log/auth.log
    maxretry: 3
  - name: nginx-http-auth
    enabled: true
    port: http,https
    logpath: /var/log/nginx/error.log
```

---

### Usuarios

#### `user_discovery` `Linux | Windows`
Lee los usuarios reales del sistema y expone el fact `discovered_users` para ser usado por otros roles.

| Variable | Defecto | Descripción |
|---|---|---|
| `user_min_uid` | `1000` | UID mínimo para considerar usuario real (Linux) |
| `user_exclude_uids` | `[65534]` | UIDs a excluir (Linux) |
| `user_exclude_shells` | nologin, false... | Shells sin acceso interactivo (Linux) |
| `user_exclude_names_windows` | Administrator, Guest... | Cuentas del sistema a excluir (Windows) |

**Resultado:** la variable `discovered_users` queda disponible para el resto de roles en el mismo play:

```yaml
discovered_users:
  - name: stevenz
    uid: 1000
    home: /home/stevenz
    shell: /bin/bash
```

---

#### `user_dirs` `Linux | Windows`
Crea una estructura de carpetas para cada usuario. Usa `$USER` como marcador posicionable en cualquier parte de la ruta.

| Variable | Defecto | Descripción |
|---|---|---|
| `user_dirs_structure` | `[/desktop/$USER/downloads, ...]` | Rutas a crear con marcador `$USER` |
| `user_dirs_mode` | `0755` | Permisos de los directorios (Linux) |

```yaml
# $USER puede ir en cualquier posición de la ruta
user_dirs_structure:
  - /desktop/$USER/downloads
  - /serv/data/$USER/local
  - /backup/$USER
```

**Requiere:** ejecutar `user_discovery` antes en el mismo play.

---

#### `user_create` `Linux | Windows`
Crea, modifica o elimina usuarios del sistema.

| Variable | Defecto | Descripción |
|---|---|---|
| `system_users` | `[]` | Lista de usuarios a gestionar |

```yaml
system_users:
  - name: deployer
    password: "{{ vault_user_hash }}"
    groups: [docker, sudo]
    shell: /bin/bash
    comment: "Usuario de despliegue"
    state: present

  - name: usuario_viejo
    state: absent
```

---

#### `user_sudo` `Linux | Windows`
Gestiona permisos elevados. Crea entradas en `sudoers.d` en Linux y gestiona el grupo Administradores en Windows.

| Variable | Defecto | Descripción |
|---|---|---|
| `sudo_users` | `[]` | Usuarios con regla sudo (Linux) |
| `sudo_groups` | `[]` | Grupos con regla sudo (Linux) |
| `sudo_windows_admin_users` | `[]` | Usuarios a añadir a Administradores (Windows) |
| `sudo_windows_group_members` | `[]` | Usuarios a añadir a grupos específicos (Windows) |

```yaml
# Linux
sudo_users:
  - name: deployer
    nopasswd: true
    commands: ALL
  - name: monitor
    nopasswd: false
    commands: /usr/bin/systemctl status *, /usr/bin/journalctl

sudo_groups:
  - name: developers
    nopasswd: false
    commands: /usr/bin/docker, /usr/bin/systemctl

# Windows
sudo_windows_admin_users:
  - deployer
sudo_windows_group_members:
  - group: "Remote Desktop Users"
    members: [deployer, monitor]
```

---

### Healthcheck

#### `healthcheck` `Linux | Windows`
Verifica que los servicios desplegados están funcionando correctamente. Si algún check falla, **el play se detiene con error** — no enmascara fallos. Se usa al final de la lista de roles en los playbooks.

Soporta tres tipos de comprobación:

| Tipo | Qué verifica | Módulo usado |
|---|---|---|
| `service` | El servicio está activo y corriendo | `service_facts` + `assert` (Linux) / `win_service_info` (Windows) |
| `port` | El puerto está abierto y escuchando | `wait_for` |
| `http` | El servidor responde con el código HTTP esperado | `uri` |

```yaml
roles:
  - role: nginx
  - role: healthcheck
    vars:
      healthchecks:
        - { type: service, name: nginx }
        - { type: port,    port: "{{ nginx_server_port }}" }
        - { type: http,    url: "http://localhost:{{ nginx_server_port }}" }
```

Para servicios con nombre distinto en Windows, usa `name_windows`:
```yaml
- { type: service, name: apache2, name_windows: Apache2.4 }
```

Para HTTP con código de respuesta distinto de 200:
```yaml
- { type: http, url: "https://mi-sitio.com/health", status_code: 204 }
```

---

### Backup

#### `backup_local` `Linux | Windows`
Crea copias de seguridad locales con fecha. Usa `rsync` + `cron` en Linux y `robocopy` + Task Scheduler en Windows.

| Variable | Defecto | Descripción |
|---|---|---|
| `backup_local_sources` | `[]` | Rutas a incluir en el backup |
| `backup_local_destination` | `/backup/local` | Destino (Linux) |
| `backup_local_destination_windows` | `C:\Backup\local` | Destino (Windows) |
| `backup_local_retention_days` | `7` | Días de retención |
| `backup_local_schedule` | `{hour: 2, minute: 0}` | Programación |

```yaml
backup_local_sources:
  - /etc
  - /var/www
  - /home

backup_local_retention_days: 14
backup_local_schedule:
  hour: "3"
  minute: "30"
  weekday: "*"
```

---

#### `backup_remote` `Linux | Windows`
Envía copias de seguridad a un servidor remoto. Usa `rsync` sobre SSH en Linux y `robocopy` a recurso UNC en Windows.

| Variable | Defecto | Descripción |
|---|---|---|
| `backup_remote_sources` | `[]` | Rutas a enviar |
| `backup_remote_host` | `""` | Servidor destino (Linux) |
| `backup_remote_user` | `backup` | Usuario SSH remoto (Linux) |
| `backup_remote_path` | `/backup/remote` | Ruta remota (Linux) |
| `backup_remote_share_windows` | `""` | Recurso UNC destino (Windows) |

```yaml
# Linux
backup_remote_sources:
  - /etc
  - /var/www
backup_remote_host: 192.168.1.50
backup_remote_user: backup
backup_remote_path: /backup/servers

# Windows
backup_remote_share_windows: "\\\\nas\\backup"
backup_remote_user_windows: backup_user
backup_remote_password_windows: "{{ vault_backup_pass }}"
```

---

## Playbooks

Cada playbook es independiente — no define variables, solo orquesta roles. Toda la configuración vive en los defaults del rol.

| Playbook | Roles que usa |
|---|---|
| `setup_common.yml` | common |
| `update_system.yml` | update |
| `setup_ssh.yml` | ssh, healthcheck |
| `setup_nginx.yml` | common, nginx, healthcheck |
| `setup_apache.yml` | common, apache, healthcheck |
| `setup_postgresql.yml` | postgresql |
| `setup_mariadb.yml` | mariadb |
| `setup_docker.yml` | docker_install, docker_daemon, docker_network, docker_volume, docker_container |
| `setup_firewall.yml` | firewall |
| `setup_fail2ban.yml` | fail2ban |
| `setup_users.yml` | user_create, user_sudo |
| `setup_user_dirs.yml` | user_discovery, user_dirs |
| `setup_backup_local.yml` | backup_local |
| `setup_backup_remote.yml` | backup_remote |

---

## Uso rápido

### Ejecutar un playbook

```bash
ansible-playbook playbooks/setup_docker.yml --ask-become-pass
```

### Limitar la ejecución a un host o grupo

```bash
ansible-playbook playbooks/setup_nginx.yml -e target_hosts=webservers --ask-become-pass
ansible-playbook playbooks/setup_postgresql.yml -e target_hosts=db01 --ask-become-pass
```

### Sobreescribir variables desde la CLI

```bash
# Cambiar el puerto SSH
ansible-playbook playbooks/setup_ssh.yml -e ssh_port=2222 --ask-become-pass

# Permitir reinicio automático tras update
ansible-playbook playbooks/update_system.yml -e update_allow_reboot=true --ask-become-pass

# Cambiar estructura de carpetas de usuario
ansible-playbook playbooks/setup_user_dirs.yml \
  -e '{"user_dirs_structure": ["/data/$USER/backup", "/logs/$USER"]}' \
  --ask-become-pass
```

### Usar solo los roles que necesitas en tu propio playbook

```yaml
# Solo Docker sin nada más
roles:
  - role: docker_install

# Docker ya instalado, solo lanzar contenedores
roles:
  - role: docker_network
  - role: docker_volume
  - role: docker_container

# Base de datos + backup
roles:
  - role: postgresql
  - role: backup_local

# Configuración completa de seguridad
roles:
  - role: firewall
  - role: fail2ban
  - role: ssh
```

### Inventario de hosts

Edita `inventory/hosts`:

```ini
[webservers]
web01 ansible_host=192.168.1.10
web02 ansible_host=192.168.1.11

[dbservers]
db01 ansible_host=192.168.1.20

[local]
localhost ansible_connection=local
```

---

## Dónde configurar las variables

Los playbooks **no contienen variables** — solo indican qué roles ejecutar. Toda la configuración vive en el rol o en los ficheros de inventario.

### Orden de precedencia (de menor a mayor)

```
roles/<rol>/defaults/main.yml     ← valores por defecto (editar aquí para cambios globales)
inventory/group_vars/all.yml      ← sobreescribe para todos los hosts
inventory/group_vars/<grupo>.yml  ← sobreescribe para un grupo concreto
inventory/host_vars/<host>.yml    ← sobreescribe para un host concreto
ansible-playbook -e "var=valor"   ← sobreescribe puntualmente desde la CLI
```

### Flujo recomendado

**1. Cambio global** — edita `roles/<rol>/defaults/main.yml`:
```
roles/ssh/defaults/main.yml          ← puerto, permisos, usuarios SSH
roles/nginx/defaults/main.yml        ← server_name, puerto, webroot
roles/postgresql/defaults/main.yml   ← versión, bases de datos, usuarios
roles/firewall/defaults/main.yml     ← reglas de firewall
```

**2. Por entorno** — si necesitas valores distintos por grupo o host:
```yaml
# inventory/group_vars/webservers.yml
nginx_server_name: mi-sitio.com
nginx_server_port: 8080

# inventory/host_vars/db01.yml
postgresql_max_connections: 200
```

**3. Puntual desde la CLI** — sin tocar ningún fichero:
```bash
ansible-playbook playbooks/setup_nginx.yml -e "nginx_server_name=ejemplo.com"
```

### Qué fichero editar según el caso

| Quiero cambiar... | Edito... |
|---|---|
| El valor por defecto de un rol para todos los hosts | `roles/<rol>/defaults/main.yml` |
| El valor solo para un grupo de hosts | `inventory/group_vars/<grupo>.yml` |
| El valor solo para un host concreto | `inventory/host_vars/<host>.yml` |
| Un valor puntual sin modificar ficheros | `-e "variable=valor"` en la CLI |

---

## Convenciones

- **`defaults/main.yml`** — variables con menor precedencia, siempre sobreescribibles.
- **`vars/main.yml`** — variables internas del rol, no pensadas para sobreescribir.
- **`group_vars/`** — variables compartidas por grupo de hosts.
- **Secretos** — usar Ansible Vault. Nunca subir contraseñas en texto plano. Los ficheros `vault*`, `secrets.yml` y similares están en `.gitignore`.
- **Colecciones** — los módulos `community.docker` requieren que Docker esté accesible en el host controlado.
