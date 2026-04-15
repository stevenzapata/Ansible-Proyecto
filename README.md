# Ansible-Proyecto

Colección de roles Ansible **reutilizables y cross-platform** (Linux + Windows) para administración de servidores e infraestructura.

---

## Índice

- [Requisitos](#requisitos)
- [Estructura del proyecto](#estructura-del-proyecto)
- [Patrón cross-platform](#patrón-cross-platform)
- [Roles disponibles](#roles-disponibles)
- [Playbooks](#playbooks)
- [Uso rápido](#uso-rápido)
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
├── ansible.cfg                     # Configuración global de Ansible
├── inventory/
│   ├── hosts                       # Inventario de hosts
│   └── group_vars/
│       └── all.yml                 # Variables para todos los hosts
├── group_vars/
│   └── all.yml                     # Variables globales extra
├── playbooks/
│   ├── update_system.yml
│   ├── setup_ssh.yml
│   ├── setup_nginx.yml
│   ├── setup_apache.yml
│   ├── setup_user_dirs.yml
│   └── setup_docker.yml
└── roles/
    ├── common/          [Linux | Windows]   Paquetes base y zona horaria
    ├── update/          [Linux | Windows]   Actualización del sistema
    ├── ssh/             [Linux | Windows]   Instalación y hardening de SSH
    ├── nginx/           [Linux | Windows]   Servidor web Nginx
    ├── apache/          [Linux | Windows]   Servidor web Apache
    ├── user_discovery/  [Linux | Windows]   Detecta usuarios reales del sistema
    ├── user_dirs/       [Linux | Windows]   Crea estructura de carpetas por usuario
    ├── docker_install/  [Linux | Windows]   Instala Docker Engine / Docker Desktop
    ├── docker_daemon/   [Linux | Windows]   Configura daemon.json
    ├── docker_network/  [Cross-platform]    Gestiona redes Docker
    ├── docker_volume/   [Cross-platform]    Gestiona volúmenes Docker
    ├── docker_container/[Cross-platform]    Despliega contenedores Docker
    ├── firewall/        [Linux | Windows]   Reglas de firewall (ufw/firewalld/WinFirewall)
    ├── fail2ban/        [Linux | Windows]   Protección fuerza bruta (fail2ban/IPBan)
    ├── postgresql/      [Linux | Windows]   Instalación y gestión de PostgreSQL
    ├── mariadb/         [Linux | Windows]   Instalación y gestión de MariaDB
    ├── backup_local/    [Linux | Windows]   Backup local (rsync/robocopy + scheduler)
    ├── backup_remote/   [Linux | Windows]   Backup remoto (rsync SSH / UNC share)
    ├── user_create/     [Linux | Windows]   Crear y gestionar usuarios del sistema
    └── user_sudo/       [Linux | Windows]   Permisos sudo/Administradores
```

Cada rol con archivos `linux.yml`/`windows.yml` sigue esta estructura interna:

```
roles/<rol>/
├── tasks/
│   ├── main.yml        ← dispatcher: detecta el SO y delega
│   ├── linux.yml       ← lógica específica Linux
│   └── windows.yml     ← lógica específica Windows
├── defaults/
│   └── main.yml        ← variables con secciones # ── Linux y # ── Windows
└── handlers/
    └── main.yml        ← handlers con condición when: ansible_os_family
```

---

## Patrón cross-platform

Todos los roles detectan automáticamente el sistema operativo del host y ejecutan las tareas correspondientes. No es necesario indicar nada manualmente.

**`tasks/main.yml`** de cada rol:

```yaml
- name: Detectar SO e incluir tareas correspondientes
  ansible.builtin.include_tasks: "{{ 'windows' if ansible_os_family == 'Windows' else 'linux' }}.yml"
```

Los roles `docker_network`, `docker_volume` y `docker_container` no necesitan split porque usan la API de Docker directamente mediante `community.docker`, que funciona igual en ambos sistemas operativos.

---

## Roles disponibles

### `common` `Linux | Windows`
Instala paquetes base y configura zona horaria y NTP.

| Variable | Defecto | Descripción |
|---|---|---|
| `common_packages` | curl, git, vim... | Paquetes a instalar (Linux) |
| `common_packages_windows` | git, curl, 7zip... | Paquetes vía Chocolatey (Windows) |
| `timezone` | `Europe/Madrid` | Zona horaria Linux |
| `timezone_windows` | `Romance Standard Time` | Zona horaria Windows |

---

### `update` `Linux | Windows`
Actualiza todos los paquetes del sistema.

| Variable | Defecto | Descripción |
|---|---|---|
| `update_upgrade_type` | `dist` | Tipo de upgrade en apt: `safe`, `full`, `dist` |
| `update_autoremove` | `true` | Eliminar paquetes huérfanos |
| `update_allow_reboot` | `false` | Reiniciar automáticamente si es necesario |

---

### `ssh` `Linux | Windows`
Instala OpenSSH Server y aplica hardening de configuración.

| Variable | Defecto | Descripción |
|---|---|---|
| `ssh_port` | `22` | Puerto de escucha |
| `ssh_permit_root_login` | `no` | Permitir login de root |
| `ssh_password_authentication` | `no` | Autenticación por contraseña |
| `ssh_users` | `[]` | Lista de `{name, public_key}` para autorizar |

```yaml
ssh_users:
  - name: deployer
    public_key: "ssh-rsa AAAA..."
```

---

### `nginx` `Linux | Windows`
Instala y configura Nginx con soporte de vhosts.

| Variable | Defecto | Descripción |
|---|---|---|
| `nginx_server_name` | `default` | Nombre del servidor / vhost |
| `nginx_server_port` | `80` | Puerto de escucha |
| `nginx_webroot` | `/var/www/html` | Directorio raíz (Linux) |
| `nginx_webroot_windows` | `C:\tools\nginx\html` | Directorio raíz (Windows) |
| `nginx_deploy_test_page` | `true` | Desplegar página de prueba |

---

### `apache` `Linux | Windows`
Instala y configura Apache con soporte de VirtualHosts.

| Variable | Defecto | Descripción |
|---|---|---|
| `apache_server_name` | `mi-sitio.local` | Nombre del VirtualHost |
| `apache_server_port` | `80` | Puerto de escucha |
| `apache_webroot` | `/var/www/html` | Directorio raíz (Linux) |
| `apache_webroot_windows` | `C:\Apache24\htdocs` | Directorio raíz (Windows) |
| `apache_modules` | rewrite, headers, ssl | Módulos a habilitar |

---

### `user_discovery` `Linux | Windows`
Lee los usuarios reales del sistema y expone el fact `discovered_users` para ser usado por otros roles.

| Variable | Defecto | Descripción |
|---|---|---|
| `user_min_uid` | `1000` | UID mínimo para considerar usuario real (Linux) |
| `user_exclude_uids` | `[65534]` | UIDs a excluir (Linux) |
| `user_exclude_shells` | nologin, false... | Shells sin acceso interactivo (Linux) |
| `user_exclude_names_windows` | Administrator, Guest... | Cuentas del sistema a excluir (Windows) |

**Resultado:** la variable `discovered_users` queda disponible para todos los roles que se ejecuten después en el mismo play:

```yaml
discovered_users:
  - name: stevenz
    uid: 1000
    home: /home/stevenz
    shell: /bin/bash
```

---

### `user_dirs` `Linux | Windows`
Crea una estructura de carpetas para cada usuario usando `$USER` como marcador posicionable en cualquier parte de la ruta.

| Variable | Defecto | Descripción |
|---|---|---|
| `user_dirs_structure` | `[/desktop/$USER/downloads, ...]` | Rutas a crear con marcador `$USER` |
| `user_dirs_mode` | `0755` | Permisos de los directorios (Linux) |

```yaml
# $USER puede ir en cualquier posición
user_dirs_structure:
  - /desktop/$USER/downloads
  - /serv/data/$USER/local
  - /backup/$USER
```

**Requiere:** ejecutar `user_discovery` antes en el mismo play.

---

### `docker_install` `Linux | Windows`
Instala Docker Engine (Linux) o Docker Desktop (Windows), arranca el servicio y añade usuarios al grupo docker.

| Variable | Defecto | Descripción |
|---|---|---|
| `docker_packages` | docker-ce, cli, compose... | Paquetes a instalar (Linux) |
| `docker_users` | `[]` | Usuarios añadidos al grupo docker |

---

### `docker_daemon` `Linux | Windows`
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

### `docker_network` `Cross-platform`
Crea y gestiona redes Docker personalizadas.

```yaml
docker_networks:
  - name: frontend
    driver: bridge
  - name: backend
    driver: bridge
    internal: true     # sin acceso externo
  - name: red-vieja
    state: absent      # eliminar si existe
```

---

### `docker_volume` `Cross-platform`
Crea y gestiona volúmenes Docker persistentes.

```yaml
docker_volumes:
  - name: datos-postgres
  - name: logs
    driver: local
    driver_options:    # montar directorio del host como volumen nombrado
      type: none
      device: /srv/logs
      o: bind
```

---

### `docker_container` `Cross-platform`
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
    labels:
      app: web
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
    state: absent      # parar y eliminar
```

| Clave | Defecto | Descripción |
|---|---|---|
| `state` | `started` | `started` / `stopped` / `absent` |
| `restart_policy` | `unless-stopped` | `always` / `unless-stopped` / `on-failure` / `no` |
| `pull` | `true` | Forzar pull de imagen al arrancar |

---

### `firewall` `Linux | Windows`
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
```

---

### `fail2ban` `Linux | Windows`
Protección contra ataques de fuerza bruta. Usa `fail2ban` en Linux e `IPBan` en Windows.

| Variable | Defecto | Descripción |
|---|---|---|
| `fail2ban_bantime` | `3600` | Segundos de baneo |
| `fail2ban_maxretry` | `5` | Intentos antes del baneo |
| `fail2ban_jails` | `[sshd]` | Jails a habilitar (Linux) |
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

### `postgresql` `Linux | Windows`
Instala PostgreSQL y gestiona bases de datos y usuarios. Requiere la colección `community.postgresql`.

| Variable | Defecto | Descripción |
|---|---|---|
| `postgresql_version` | `16` | Versión a instalar |
| `postgresql_port` | `5432` | Puerto de escucha |
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

### `mariadb` `Linux | Windows`
Instala MariaDB, aplica securización inicial y gestiona bases de datos y usuarios. Requiere `community.mysql`.

| Variable | Defecto | Descripción |
|---|---|---|
| `mariadb_root_password` | `""` | Contraseña de root (usar Vault) |
| `mariadb_port` | `3306` | Puerto de escucha |
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

### `backup_local` `Linux | Windows`
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

### `backup_remote` `Linux | Windows`
Envía copias de seguridad a un servidor remoto. Usa `rsync` sobre SSH en Linux y `robocopy` a recurso UNC en Windows.

| Variable | Defecto | Descripción |
|---|---|---|
| `backup_remote_sources` | `[]` | Rutas a enviar |
| `backup_remote_host` | `""` | Servidor destino (Linux) |
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

### `user_create` `Linux | Windows`
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

### `user_sudo` `Linux | Windows`
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

## Playbooks

| Playbook | Roles que usa |
|---|---|
| `update_system.yml` | update |
| `setup_ssh.yml` | ssh |
| `setup_nginx.yml` | nginx |
| `setup_apache.yml` | apache |
| `setup_user_dirs.yml` | user_discovery, user_dirs |
| `setup_docker.yml` | docker_install, docker_daemon, docker_network, docker_volume, docker_container |

---

## Uso rápido

### Ejecutar un playbook completo

```bash
ansible-playbook playbooks/setup_docker.yml --ask-become-pass
```

### Usar solo los roles que necesitas

```yaml
# Solo instalar Docker sin desplegar nada
roles:
  - role: docker_install

# Docker ya instalado, solo lanzar contenedores
roles:
  - role: docker_network
  - role: docker_volume
  - role: docker_container

# Crear carpetas para usuarios del sistema
roles:
  - role: user_discovery
  - role: user_dirs
```

### Sobreescribir variables desde la CLI

```bash
# Cambiar estructura de carpetas de usuario
ansible-playbook playbooks/setup_user_dirs.yml \
  -e '{"user_dirs_structure": ["/data/$USER/backup", "/logs/$USER"]}' \
  --ask-become-pass

# Permitir reinicio automático tras update
ansible-playbook playbooks/update_system.yml \
  -e update_allow_reboot=true \
  --ask-become-pass
```

### Inventario de hosts

Edita `inventory/hosts` con las IPs de tus servidores:

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

Los playbooks **no contienen variables** — solo indican qué roles ejecutar y en qué hosts. Toda la configuración vive en el rol o en los ficheros de inventario.

### Orden de precedencia (de menor a mayor)

```
roles/<rol>/defaults/main.yml   ← valores por defecto del rol (editar aquí para cambios globales)
inventory/group_vars/all.yml    ← sobreescribe para todos los hosts
inventory/group_vars/<grupo>.yml← sobreescribe para un grupo concreto
inventory/host_vars/<host>.yml  ← sobreescribe para un host concreto
ansible-playbook -e "var=valor" ← sobreescribe puntualmente desde la CLI
```

### Flujo recomendado

**1. Configurar el rol** — edita `roles/<rol>/defaults/main.yml`:
```
roles/
  ssh/defaults/main.yml         ← puerto, permisos, usuarios SSH
  nginx/defaults/main.yml       ← server_name, puerto, webroot
  postgresql/defaults/main.yml  ← versión, bases de datos, usuarios
  firewall/defaults/main.yml    ← reglas de firewall
  ...
```

**2. Sobreescribir por grupo o host** — si necesitas valores distintos por entorno:
```yaml
# inventory/group_vars/webservers.yml
nginx_server_name: mi-sitio.com
nginx_server_port: 8080

# inventory/host_vars/web01.yml
nginx_deploy_test_page: false
```

**3. Sobreescribir puntualmente desde la CLI** — sin tocar ningún fichero:
```bash
ansible-playbook playbooks/setup_nginx.yml -e "nginx_server_name=ejemplo.com"
ansible-playbook playbooks/update_system.yml -e "update_allow_reboot=true"
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
