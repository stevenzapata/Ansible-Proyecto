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
│   ├── site.yml                    # Playbook maestro
│   ├── update_system.yml
│   ├── setup_ssh.yml
│   ├── setup_nginx.yml
│   ├── setup_apache.yml
│   ├── setup_user_dirs.yml
│   └── setup_docker.yml
└── roles/
    ├── common/                     # Paquetes base y zona horaria
    ├── update/                     # Actualización del sistema
    ├── ssh/                        # Instalación y hardening de SSH
    ├── nginx/                      # Servidor web Nginx
    ├── apache/                     # Servidor web Apache
    ├── user_discovery/             # Detecta usuarios reales del sistema
    ├── user_dirs/                  # Crea estructura de carpetas por usuario
    ├── docker_install/             # Instala Docker Engine / Docker Desktop
    ├── docker_daemon/              # Configura daemon.json
    ├── docker_network/             # Gestiona redes Docker
    ├── docker_volume/              # Gestiona volúmenes Docker
    └── docker_container/           # Despliega contenedores Docker
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

### `common`
Instala paquetes base y configura zona horaria y NTP.

| Variable | Defecto | Descripción |
|---|---|---|
| `common_packages` | curl, git, vim... | Paquetes a instalar (Linux) |
| `common_packages_windows` | git, curl, 7zip... | Paquetes vía Chocolatey (Windows) |
| `timezone` | `Europe/Madrid` | Zona horaria Linux |
| `timezone_windows` | `Romance Standard Time` | Zona horaria Windows |

---

### `update`
Actualiza todos los paquetes del sistema.

| Variable | Defecto | Descripción |
|---|---|---|
| `update_upgrade_type` | `dist` | Tipo de upgrade en apt: `safe`, `full`, `dist` |
| `update_autoremove` | `true` | Eliminar paquetes huérfanos |
| `update_allow_reboot` | `false` | Reiniciar automáticamente si es necesario |

---

### `ssh`
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

### `nginx`
Instala y configura Nginx con soporte de vhosts.

| Variable | Defecto | Descripción |
|---|---|---|
| `nginx_server_name` | `default` | Nombre del servidor / vhost |
| `nginx_server_port` | `80` | Puerto de escucha |
| `nginx_webroot` | `/var/www/html` | Directorio raíz (Linux) |
| `nginx_webroot_windows` | `C:\tools\nginx\html` | Directorio raíz (Windows) |
| `nginx_deploy_test_page` | `true` | Desplegar página de prueba |

---

### `apache`
Instala y configura Apache con soporte de VirtualHosts.

| Variable | Defecto | Descripción |
|---|---|---|
| `apache_server_name` | `mi-sitio.local` | Nombre del VirtualHost |
| `apache_server_port` | `80` | Puerto de escucha |
| `apache_webroot` | `/var/www/html` | Directorio raíz (Linux) |
| `apache_webroot_windows` | `C:\Apache24\htdocs` | Directorio raíz (Windows) |
| `apache_modules` | rewrite, headers, ssl | Módulos a habilitar |

---

### `user_discovery`
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

### `user_dirs`
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

### `docker_install`
Instala Docker Engine (Linux) o Docker Desktop (Windows), arranca el servicio y añade usuarios al grupo docker.

| Variable | Defecto | Descripción |
|---|---|---|
| `docker_packages` | docker-ce, cli, compose... | Paquetes a instalar (Linux) |
| `docker_users` | `[]` | Usuarios añadidos al grupo docker |

---

### `docker_daemon`
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

### `docker_network`
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

### `docker_volume`
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

### `docker_container`
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

## Playbooks

| Playbook | Roles que usa |
|---|---|
| `site.yml` | common, update, ssh, nginx |
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

## Convenciones

- **`defaults/main.yml`** — variables con menor precedencia, siempre sobreescribibles.
- **`vars/main.yml`** — variables internas del rol, no pensadas para sobreescribir.
- **`group_vars/`** — variables compartidas por grupo de hosts.
- **Secretos** — usar Ansible Vault. Nunca subir contraseñas en texto plano. Los ficheros `vault*`, `secrets.yml` y similares están en `.gitignore`.
- **Colecciones** — los módulos `community.docker` requieren que Docker esté accesible en el host controlado.
