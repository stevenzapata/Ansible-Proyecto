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
  - [Infraestructura](#infraestructura)
- [Playbooks](#playbooks)
- [Uso rápido](#uso-rápido)
- [Dónde configurar las variables](#dónde-configurar-las-variables)
- [Gestión de secretos con Ansible Vault](#gestión-de-secretos-con-ansible-vault)
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

### Requisitos para hosts Windows Server

**WinRM viene habilitado por defecto en Windows Server** (2019/2022/2025) — no hace falta configurarlo manualmente. Ansible conecta directamente con NTLM en cuanto la máquina está en red.

El único requisito previo es habilitar la virtualización anidada en el hipervisor si se va a usar Hyper-V isolation para contenedores Docker (ver sección [Hyper-V isolation](#hyper-v-isolation-en-vms)):

- **Proxmox**: `qm set <vmid> -cpu host` (con la VM apagada)
- **vSphere**: activar *Expose hardware-assisted virtualization* en la configuración de CPU de la VM

### Colecciones necesarias

```bash
ansible-galaxy collection install \
  ansible.posix \
  community.general \
  community.crypto \
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
│       └── all/
│           ├── main.yml    ← variables globales
│           └── vault.yml   ← secretos cifrados con Ansible Vault
├── playbooks/
│   ├── setup_common.yml
│   ├── update_system.yml
│   ├── setup_ssh.yml
│   ├── linux_setup_ssh.yml
│   ├── windows_setup_ssh.yml
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
│   ├── setup_backup_remote.yml
│   └── snapshot_and_update.yml
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
    ├── docker_network/   [Linux | Windows]
    ├── docker_volume/    [Linux | Windows]
    ├── docker_container/ [Linux | Windows]
    ├── firewall/         [Linux | Windows]
    ├── fail2ban/         [Linux | Windows]
    ├── user_discovery/   [Linux | Windows]
    ├── user_dirs/        [Linux | Windows]
    ├── user_create/      [Linux | Windows]
    ├── user_sudo/        [Linux | Windows]
    ├── backup_local/     [Linux | Windows]
    ├── backup_remote/    [Linux | Windows]
    ├── ssh_keygen/       [Linux | Windows]
    ├── healthcheck/      [Linux | Windows]
    └── proxmox_snapshot/ [Linux | Windows]
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
  ansible.builtin.include_tasks:
    file: "{{ 'windows' if ansible_os_family == 'Windows' else 'linux' }}.yml"
```

Los roles `docker_network`, `docker_volume` y `docker_container` tienen split `linux.yml`/`windows.yml`. En Linux usan los módulos `community.docker.*`; en Windows usan `ansible.windows.win_powershell` con la CLI de Docker directamente, ya que los módulos Python de `community.docker` no están disponibles en Windows.

---

## Roles por categoría

---

### Sistema

#### `common` `Linux | Windows`
Instala paquetes base y configura zona horaria y NTP.

| Variable | Defecto | Descripción |
|---|---|---|
| `common_packages` | curl, git, vim... | Paquetes a instalar (Linux) |
| `timezone` | `Europe/Madrid` | Zona horaria Linux |
| `common_packages_windows` | git, vim, curl, 7zip... | Paquetes vía Chocolatey (Windows) |
| `timezone_windows` | `Romance Standard Time` | Zona horaria Windows |
| `common_ntp_servers_windows` | `[0.pool.ntp.org, 1.pool.ntp.org]` | Servidores NTP (Windows) |

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

#### `ssh_keygen` `Linux | Windows`
Genera un par de claves SSH (pública + privada) para un usuario del sistema. Pensado para que servidores como el de backup puedan conectarse a otros servidores sin contraseña. La clave pública se expone como el fact `ssh_keygen_public_key` para poder registrarla donde sea necesario.

| Variable | Defecto | Descripción |
|---|---|---|
| `ssh_keygen_user` | `root` | Usuario para el que se genera la clave |
| `ssh_keygen_type` | `ed25519` | Tipo de clave: `ed25519`, `rsa`, `ecdsa` |
| `ssh_keygen_comment` | `ansible-managed` | Comentario identificativo en la clave pública |
| `ssh_keygen_passphrase` | `""` | Passphrase — vacío para uso en cron/scripts |
| `ssh_keygen_force` | `false` | Regenerar aunque ya exista |
| `ssh_keygen_path` | `<home>/.ssh/id_<type>` | Ruta personalizada (Linux) |
| `ssh_keygen_path_windows` | `C:\Users\<user>\.ssh\id_<type>` | Ruta personalizada (Windows) |

Al finalizar imprime la clave pública con instrucciones para registrarla en el servidor destino. Requiere la colección `community.crypto` (Linux).

---

#### `ssh` `Linux | Windows`
Configura el acceso SSH seguro. En Linux aplica hardening desplegando `sshd_config` desde plantilla (con validación previa al reinicio) y gestiona `authorized_keys` por usuario — asume que el servidor SSH ya está instalado. En Windows instala la feature opcional OpenSSH Server, configura la regla de firewall, establece PowerShell como shell por defecto y gestiona `authorized_keys` tanto para usuarios normales como para administradores.

> **Importante:** el rol deshabilita la autenticación por contraseña por defecto. Si `ssh_users` está vacío al ejecutar, el rol fallará con error antes de aplicar ningún cambio — esto es intencionado para evitar quedarse sin acceso al servidor. Define siempre al menos un usuario con su clave pública antes de ejecutar.

**Variables Linux**

| Variable | Defecto | Descripción |
|---|---|---|
| `ssh_service_name` | `ssh` | Nombre del servicio (`sshd` en RedHat) |
| `ssh_port` | `22` | Puerto de escucha |
| `ssh_permit_root_login` | `no` | Permitir login de root |
| `ssh_password_authentication` | `no` | Autenticación por contraseña |
| `ssh_pubkey_authentication` | `yes` | Autenticación por clave pública |
| `ssh_max_auth_tries` | `3` | Intentos máximos de autenticación |
| `ssh_client_alive_interval` | `300` | Segundos entre keepalives al cliente |
| `ssh_client_alive_count_max` | `2` | Keepalives sin respuesta antes de desconectar |
| `ssh_allow_tcp_forwarding` | `no` | Permitir reenvío de puertos TCP |
| `ssh_x11_forwarding` | `no` | Permitir reenvío de X11 |
| `ssh_users` | `[]` | Lista de `{name, public_key}` para autorizar |

**Variables Windows**

| Variable | Defecto | Descripción |
|---|---|---|
| `windows_ssh_port` | `22` | Puerto de escucha |
| `windows_ssh_default_shell` | `powershell.exe` | Shell por defecto para sesiones SSH |
| `windows_ssh_local_public_key` | `~/.ssh/id_ed25519.pub` | Ruta local a la clave pública a copiar |
| `windows_ssh_admin_user` | `true` | Si `true`, escribe en `administrators_authorized_keys` con permisos restringidos |

```yaml
# Autorizar claves en Linux
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
| `nginx_deploy_test_page` | `true` | Desplegar página de prueba |
| `nginx_ssl_enabled` | `false` | Activar HTTPS |
| `nginx_ssl_port` | `443` | Puerto SSL |
| `nginx_webroot` | `/var/www/html` | Directorio raíz (Linux) |
| `nginx_ssl_cert` | `/etc/ssl/certs/nginx.crt` | Ruta al certificado (Linux) |
| `nginx_ssl_key` | `/etc/ssl/private/nginx.key` | Ruta a la clave privada (Linux) |
| `nginx_user` | `www-data` | Usuario del proceso nginx (Linux) |
| `nginx_worker_processes` | `auto` | Número de workers (Linux) |
| `nginx_worker_connections` | `1024` | Conexiones por worker (Linux) |
| `nginx_webroot_windows` | `C:\tools\nginx\html` | Directorio raíz (Windows) |
| `nginx_conf_dir_windows` | `C:\tools\nginx\conf` | Directorio de configuración (Windows) |
| `nginx_ssl_cert_windows` | `C:\tools\nginx\conf\ssl\nginx.crt` | Ruta al certificado (Windows) |
| `nginx_ssl_key_windows` | `C:\tools\nginx\conf\ssl\nginx.key` | Ruta a la clave privada (Windows) |

---

#### `apache` `Linux | Windows`
Instala y configura Apache con soporte de VirtualHosts.

| Variable | Defecto | Descripción |
|---|---|---|
| `apache_server_name` | `mi-sitio.local` | Nombre del VirtualHost |
| `apache_server_admin` | `admin@mi-sitio.local` | Email del administrador |
| `apache_server_port` | `80` | Puerto de escucha |
| `apache_modules` | rewrite, headers, ssl | Módulos a habilitar |
| `apache_disable_default_site` | `true` | Deshabilitar el vhost por defecto |
| `apache_deploy_test_page` | `true` | Desplegar página de prueba |
| `apache_ssl_enabled` | `false` | Activar HTTPS |
| `apache_ssl_port` | `443` | Puerto SSL |
| `apache_webroot` | `/var/www/html` | Directorio raíz (Linux) |
| `apache_ssl_cert` | `/etc/ssl/certs/apache.crt` | Ruta al certificado (Linux) |
| `apache_ssl_key` | `/etc/ssl/private/apache.key` | Ruta a la clave privada (Linux) |
| `apache_webroot_windows` | `C:\Apache24\htdocs` | Directorio raíz (Windows) |
| `apache_service_name_windows` | `Apache2.4` | Nombre del servicio (Windows) |
| `apache_vhost_dir_windows` | `C:\Apache24\conf\extra` | Directorio de vhosts (Windows) |
| `apache_ssl_cert_windows` | `C:\Apache24\conf\ssl\apache.crt` | Ruta al certificado (Windows) |
| `apache_ssl_key_windows` | `C:\Apache24\conf\ssl\apache.key` | Ruta a la clave privada (Windows) |

---

### Bases de datos

#### `postgresql` `Linux | Windows`
Instala PostgreSQL y gestiona bases de datos y usuarios. Requiere `community.postgresql`.

| Variable | Defecto | Descripción |
|---|---|---|
| `postgresql_version` | `16` | Versión a instalar |
| `postgresql_port` | `5432` | Puerto de escucha |
| `postgresql_databases` | `[]` | Bases de datos a crear |
| `postgresql_users` | `[]` | Usuarios a crear |
| `postgresql_listen_addresses` | `localhost` | Interfaces de escucha (Linux) |
| `postgresql_max_connections` | `100` | Conexiones máximas (Linux) |
| `postgresql_service_name_windows` | `postgresql-x64-<version>` | Nombre del servicio (Windows) |
| `postgresql_data_dir_windows` | `C:\Program Files\PostgreSQL\<version>\data` | Directorio de datos (Windows) |

```yaml
postgresql_databases:
  - name: miapp
    owner: appuser

postgresql_users:
  - name: appuser
    password: "{{ vault_db_password }}"   # definir vault_db_password en inventory/group_vars/all/vault.yml
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
| `mariadb_service_name_windows` | `MySQL` | Nombre del servicio (Windows) |
| `mariadb_install_dir_windows` | `C:\Program Files\MariaDB` | Directorio de instalación (Windows) |

```yaml
mariadb_root_password: "{{ vault_mysql_root }}"   # definir en inventory/group_vars/all/vault.yml

mariadb_databases:
  - name: miapp
    encoding: utf8mb4

mariadb_users:
  - name: appuser
    password: "{{ vault_db_password }}"            # definir en inventory/group_vars/all/vault.yml
    host: localhost
    priv: "miapp.*:ALL"
```

---

### Docker

Los roles Docker están pensados para usarse en cadena: primero instalar, luego configurar el daemon y finalmente gestionar redes, volúmenes y contenedores de forma independiente.

#### `docker_install` `Linux | Windows`
Instala Docker Engine y arranca el servicio.

> **Windows:** requiere **Windows Server 2019, 2022 o 2025**. Habilita la feature `Containers`, instala Docker Engine vía el script oficial de Microsoft (`install-docker-ce.ps1`) y registra el servicio `docker`. Solo soporta Windows Server — no compatible con Windows 10/11.

| Variable | Defecto | Descripción |
|---|---|---|
| `docker_packages` | docker-ce, cli, compose... | Paquetes a instalar (Linux) |
| `docker_users` | `[]` | Usuarios añadidos al grupo docker |
| `docker_hyperv_isolation` | `false` | Windows: instala la feature Hyper-V. Necesario en VMs para usar `isolation: hyperv` en contenedores |
| `docker_allow_reboot` | `false` | Reiniciar automáticamente si la instalación de features lo requiere |

---

> **Windows en VM — Hyper-V isolation y virtualización anidada**
>
> Los contenedores Windows tienen dos modos de ejecución:
>
> | Modo | Cómo funciona | Requisito |
> |---|---|---|
> | **Process isolation** (defecto) | El contenedor comparte el kernel del host | La imagen debe haberse compilado para el mismo build exacto del host |
> | **Hyper-V isolation** | Cada contenedor arranca en una VM ligera | La VM host necesita nested virtualization habilitada en el hipervisor |
>
> **Por qué process isolation falla en VMs actualizadas:** Microsoft publica nuevas imágenes de contenedor Windows cada Patch Tuesday. Si el host recibe actualizaciones de Windows y la imagen no coincide exactamente con el build del kernel, el contenedor falla con error `0xc0370106` o `0xc0370112`. Hyper-V isolation elimina esta restricción — el contenedor corre en su propia VM ligera independientemente de la versión del host.
>
> Para usar Hyper-V isolation hay que habilitar la feature de Windows y la virtualización anidada en el hipervisor.
>
> **Paso 1 — Habilitar nested virtualization en el hipervisor**
>
> *Proxmox* — la VM debe estar apagada:
> ```bash
> # Opción 1: desde la web UI
> # VM → Hardware → Processors → Type: seleccionar "host" o "x86-64-v2-AES"
>
> # Opción 2: desde la CLI del nodo Proxmox
> qm set <vmid> -cpu host
> qm start <vmid>
>
> # Opción 3: editar directamente el fichero de configuración
> # Añadir o modificar la línea: cpu: host
> nano /etc/pve/qemu-server/<vmid>.conf
> ```
>
> *vSphere* — la VM debe estar apagada:
> ```
> # Opción 1: desde la UI
> Edit Settings → CPU → activar "Expose hardware assisted virtualization to the guest OS"
>
> # Opción 2: vía PowerCLI
> ```
> ```powershell
> $vm = Get-VM -Name "NombreVM"
> $spec = New-Object VMware.Vim.VirtualMachineConfigSpec
> $spec.nestedHVEnabled = $true
> $vm.ExtensionData.ReconfigVM($spec)
> ```
>
> **Paso 2 — Activar `docker_hyperv_isolation` en el inventario:**
>
> El rol `docker_install` instala la feature Hyper-V automáticamente cuando esta variable está activa. En `group_vars/windows.yml`:
> ```yaml
> docker_hyperv_isolation: true
> docker_allow_reboot: true   # reinicia si la instalación lo requiere
> ```
> A continuación ejecuta el playbook `setup_docker.yml` — no hace falta ningún comando manual.
>
> **Paso 3 — Configurar el contenedor con `isolation: hyperv`** (ver sección [`docker_container`](#docker_container-linux--windows)).

---

#### `docker_daemon` `Linux | Windows`
Configura el daemon de Docker mediante `daemon.json`.

| Variable | Defecto | Descripción |
|---|---|---|
| `docker_daemon_options` | log-driver, storage-driver... | Dict completo para `daemon.json` |
| `docker_daemon_config_dir_windows` | `%ProgramData%\Docker\config` | Directorio de configuración del daemon (Windows) |

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

#### `docker_network` `Linux | Windows`
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

#### `docker_volume` `Linux | Windows`
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

#### `docker_container` `Linux | Windows`
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
| `isolation` | *(no definido)* | Solo Windows: `process` o `hyperv`. Usar `hyperv` en VMs — process isolation requiere que el build del host coincida exactamente con el de la imagen. |

---

> **Imágenes compatibles por SO**
>
> Docker Engine en Windows Server corre en **Windows containers mode** — solo puede ejecutar imágenes compiladas para Windows. La mayoría de imágenes públicas (nginx, postgres, redis, mysql, etc.) son **Linux-only** y no funcionan en un host Windows.
>
> | Host | Imágenes válidas | Ejemplos |
> |---|---|---|
> | Linux | Linux | `nginx`, `postgres`, `redis`, `mysql`, `mariadb`... |
> | Windows Server | Windows Server Core | `mcr.microsoft.com/windows/servercore/iis`, `mcr.microsoft.com/mssql/server` (Windows) |
>
> **¿Por qué no se pueden correr imágenes Linux en Windows Server?** Docker Engine en Windows Server no tiene backend Linux. LCOW (Linux Containers on Windows) existió como feature experimental pero fue abandonada — solo funciona en Docker Desktop (Windows 10/11) mediante WSL2, que no está disponible en Windows Server.
>
> El patrón correcto es usar cada host para lo que está diseñado: Linux containers en hosts Linux, Windows containers en hosts Windows. Las variables `group_vars/linux.yml` y `group_vars/windows.yml` están separadas precisamente por esta razón.

---

### Seguridad

#### `firewall` `Linux | Windows`
Gestiona reglas de firewall. Usa `ufw` en Debian/Ubuntu, `firewalld` en RedHat/CentOS y Windows Firewall en Windows.

| Variable | Defecto | Descripción |
|---|---|---|
| `firewall_rules` | `[]` | Lista de reglas a aplicar |
| `firewall_default_input` | `deny` | Política por defecto de entrada (Linux) |
| `firewall_default_output` | `allow` | Política por defecto de salida (Linux) |
| `firewall_default_forward` | `deny` | Política por defecto de forward (Linux) |
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
| `fail2ban_bantime` | `3600` | Segundos de baneo (Linux) |
| `fail2ban_maxretry` | `5` | Intentos antes del baneo (Linux) |
| `fail2ban_findtime` | `600` | Ventana de tiempo para contar intentos (Linux) |
| `fail2ban_jails` | `[sshd]` | Jails a habilitar (Linux) |
| `ipban_bantime` | `3600` | Segundos de baneo (Windows) |
| `ipban_maxretry` | `5` | Intentos antes del baneo (Windows) |
| `ipban_install_dir_windows` | `C:\Program Files\IPBan` | Directorio de instalación IPBan (Windows) |

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
| `backup_local_retention_days` | `7` | Días de retención |
| `backup_local_schedule` | `{hour: 2, minute: 0, weekday: *}` | Programación |
| `backup_local_destination` | `/backup/local` | Destino (Linux) |
| `backup_local_rsync_opts` | `--archive --compress --delete --quiet` | Opciones rsync (Linux) |
| `backup_local_destination_windows` | `C:\Backup\local` | Destino (Windows) |
| `backup_local_robocopy_opts` | `/E /COPYALL /R:3 /W:5 /NP /NFL /NDL` | Opciones robocopy (Windows) |

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
| `backup_remote_port` | `22` | Puerto SSH (Linux) |
| `backup_remote_path` | `/backup/remote` | Ruta remota (Linux) |
| `backup_remote_ssh_key` | `~/.ssh/id_ed25519` | Clave SSH para la conexión (Linux) |
| `backup_remote_rsync_opts` | `--archive --compress --delete --quiet` | Opciones rsync (Linux) |
| `backup_remote_schedule` | `{hour: 3, minute: 0, weekday: *}` | Programación (Linux) |
| `backup_remote_share_windows` | `""` | Recurso UNC destino (Windows) |
| `backup_remote_user_windows` | `""` | Usuario para el recurso UNC (Windows) |
| `backup_remote_password_windows` | `""` | Contraseña UNC — usar Vault (Windows) |
| `backup_remote_robocopy_opts` | `/E /COPYALL /R:3 /W:5 /NP /NFL /NDL` | Opciones robocopy (Windows) |

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

### Infraestructura

#### `proxmox_snapshot` `Linux | Windows`
Crea un snapshot de la VM en Proxmox antes de cualquier operación. Consulta la API REST de Proxmox desde `localhost` (el contenedor Ansible), localiza la VM por nombre coincidente con el host de Ansible, crea el snapshot y espera a que termine. Si el snapshot falla, el playbook se detiene y no continúa.

| Variable | Defecto | Descripción |
|---|---|---|
| `proxmox_host` | `192.168.169.115` | IP del nodo Proxmox |
| `proxmox_port` | `8006` | Puerto de la API REST |
| `proxmox_user` | `root@pam` | Usuario de autenticación |
| `proxmox_token_name` | `gestor-vms` | Nombre del API token |
| `proxmox_token_secret` | `{{ vault_proxmox_token_secret }}` | Secreto del token — definir en vault |
| `proxmox_node` | `pve-nodo1` | Nombre del nodo Proxmox |
| `proxmox_snapshot_prefix` | `ansible_pre-update` | Prefijo del snapshot — el nombre final incluye la fecha: `ansible_pre-update-2026-05-18` |
| `proxmox_snapshot_description` | `Snapshot automático pre-actualización — Ansible` | Descripción del snapshot |
| `proxmox_task_retries` | `24` | Intentos de polling para esperar al snapshot |
| `proxmox_task_delay` | `5` | Segundos entre cada intento de polling |

> El nombre del host en Ansible (`inventory_hostname`) debe coincidir exactamente con el nombre de la VM en Proxmox para que el rol localice el VMID automáticamente.

---

## Playbooks

Cada playbook es independiente — no define variables, solo orquesta roles. Toda la configuración vive en los defaults del rol.

| Playbook | Roles que usa | Notas |
|---|---|---|
| `setup_common.yml` | common | |
| `update_system.yml` | update | |
| `setup_ssh.yml` | ssh, healthcheck | Requiere acceso SSH ya configurado |
| `linux_setup_ssh.yml` | ssh | Bootstrap inicial en Linux — ver uso más abajo |
| `windows_setup_ssh.yml` | ssh | Bootstrap inicial en Windows vía WinRM — ver uso más abajo |
| `setup_nginx.yml` | common, nginx, healthcheck | |
| `setup_apache.yml` | common, apache, healthcheck | |
| `setup_postgresql.yml` | postgresql | |
| `setup_mariadb.yml` | mariadb | |
| `setup_docker.yml` | docker_install, docker_daemon, docker_network, docker_volume, docker_container | |
| `setup_firewall.yml` | firewall | |
| `setup_fail2ban.yml` | fail2ban | |
| `setup_users.yml` | user_create, user_sudo | |
| `setup_user_dirs.yml` | user_discovery, user_dirs | |
| `setup_backup_local.yml` | backup_local | |
| `setup_backup_remote.yml` | backup_remote | |
| `setup_ssh_keygen.yml` | ssh_keygen | |
| `snapshot_and_update.yml` | proxmox_snapshot, update | Snapshot en Proxmox antes de actualizar — aborta si el snapshot falla |

---

### Ejecución de `setup_ssh.yml`

Aplica hardening SSH y copia claves públicas a los hosts. **Requiere definir `ssh_users` antes de ejecutar** o el rol fallará al intentar deshabilitar el acceso por contraseña sin ninguna clave configurada.

```bash
# Ejecución estándar contra todos los hosts del inventario
ansible-playbook playbooks/setup_ssh.yml -i inventory/ --ask-become-pass

# Solo contra un host concreto
ansible-playbook playbooks/setup_ssh.yml -i inventory/ -e "target_hosts=CliLin" --ask-become-pass
```

**Flags:**

| Flag | Por qué se usa |
|---|---|
| `-i inventory/` | Indica el directorio de inventario. Ansible carga `hosts`, `group_vars/` y `host_vars/` desde ahí. |
| `--ask-become-pass` / `-K` | El rol usa `become: true` para ejecutar tareas como root (editar `/etc/ssh/sshd_config`, gestionar servicios). Sin este flag falla si el usuario requiere contraseña para sudo. |
| `-e "target_hosts=CliLin"` | Limita la ejecución a un host o grupo concreto. Sin él ataca el valor por defecto del playbook (normalmente `all`). |

---

### Ejecución de `setup_docker.yml`

Instala Docker, configura el daemon y despliega redes, volúmenes y contenedores. Las contraseñas de contenedores (como `POSTGRES_PASSWORD`) deben estar en un fichero Ansible Vault.

```bash
# Ejecución estándar
docker exec -it ansible ansible-playbook playbooks/setup_docker.yml -i inventory/ --ask-become-pass

# Solo contra un host concreto
docker exec -it ansible ansible-playbook playbooks/setup_docker.yml -i inventory/ -e "target_hosts=CliLin" --ask-become-pass
```

**Flags:**

| Flag | Por qué se usa |
|---|---|
| `-i inventory/` | Indica el directorio de inventario. Carga variables de `group_vars/all/main.yml` y `group_vars/all/vault.yml`. |
| `--ask-become-pass` / `-K` | Instalar Docker y gestionar servicios requiere privilegios de root. |
| `-e "target_hosts=CliLin"` | Limita la ejecución a un host o grupo concreto. |

> El vault se descifra automáticamente gracias a `vault_password_file` en `ansible.cfg`. No hace falta `--ask-vault-pass`.

**Dónde definir los contenedores a desplegar:**

Edita `inventory/group_vars/linux.yml` o `inventory/group_vars/windows.yml` según el SO del host y rellena `docker_networks`, `docker_volumes` y `docker_containers`. Consulta la sección [`docker_container`](#docker_container-linux--windows) para ver todos los parámetros disponibles.

Para el 90% de casos de uso (nginx, postgres, redis, mariadb, apps web, etc.) funciona directamente cambiando solo las variables — no hay que tocar el rol ni el playbook.

**Secretos en contenedores:** si tu contenedor necesita contraseñas (ej. `POSTGRES_PASSWORD`), defínelas en el vault y referencíalas con `{{ vault_nombre_variable }}`. Consulta la sección [Gestión de secretos con Ansible Vault](#gestión-de-secretos-con-ansible-vault).

---

## Uso rápido

### Ejecutar un playbook

```bash
ansible-playbook playbooks/setup_docker.yml -i inventory/ --ask-become-pass
```

Para limitar la ejecución a un host o grupo concreto usa `-e target_hosts=`:
```bash
ansible-playbook playbooks/setup_nginx.yml -i inventory/ -e target_hosts=webservers --ask-become-pass
```

### Bootstrap SSH (primera vez, sin clave previa)

Usa estos playbooks cuando el host aún no tiene SSH con clave configurado. Una vez ejecutados, el acceso futuro se hace con clave (sin contraseña).

**Linux** — requiere acceso inicial por contraseña:
```bash
# Copia la clave pública local (~/.ssh/id_ed25519.pub) al usuario remoto
ansible-playbook playbooks/linux_setup_ssh.yml --ask-pass --ask-become-pass

# Usar una clave pública diferente
ansible-playbook playbooks/linux_setup_ssh.yml \
  -e "linux_ssh_local_public_key=~/.ssh/id_rsa.pub" \
  --ask-pass --ask-become-pass
```

**Windows** — requiere WinRM habilitado en el host destino:

> El proyecto usa WinRM para toda la comunicación con hosts Windows. Este playbook es opcional: instala OpenSSH Server en el host Windows por si se quiere conectar por SSH en lugar de WinRM, pero no es necesario para el resto de playbooks.

```bash
# Instala OpenSSH Server y copia la clave pública local
ansible-playbook playbooks/windows_setup_ssh.yml --ask-pass

# Usar una clave pública diferente
ansible-playbook playbooks/windows_setup_ssh.yml \
  -e "windows_ssh_local_public_key=~/.ssh/id_rsa.pub" \
  --ask-pass
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
localhost ansible_connection=local ansible_python_interpreter=/usr/local/bin/python3
```

> `ansible_python_interpreter` en `localhost` apunta a `/usr/local/bin/python3` porque es la ruta dentro del contenedor Ansible. Los hosts Linux usan `/usr/bin/python3` definido en `[all:vars]`.

---

## Dónde configurar las variables

Los playbooks **no contienen variables** — solo indican qué roles ejecutar. Toda la configuración vive en el rol o en los ficheros de inventario.

### Orden de precedencia (de menor a mayor)

```
roles/<rol>/defaults/main.yml          ← valores por defecto (editar aquí para cambios globales)
inventory/group_vars/all/main.yml      ← sobreescribe para todos los hosts
inventory/group_vars/all/vault.yml     ← secretos cifrados (Ansible Vault)
inventory/group_vars/<grupo>.yml       ← sobreescribe para un grupo concreto
inventory/host_vars/<host>.yml         ← sobreescribe para un host concreto
ansible-playbook -e "var=valor"        ← sobreescribe puntualmente desde la CLI
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

Un ejemplo real del proyecto: todos los hosts Windows usan NTLM (`group_vars/windows.yml`), pero CliWin requiere autenticación básica, por lo que se sobreescribe solo para ese host:
```yaml
# inventory/host_vars/CliWin.yml
ansible_winrm_transport: basic
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

## Gestión de secretos con Ansible Vault

Todos los secretos del proyecto (contraseñas, tokens, credenciales) se almacenan cifrados en un único fichero:

```
inventory/group_vars/all/vault.yml
```

Al estar en `group_vars/all/`, Ansible lo carga automáticamente para todos los hosts en cada ejecución. Los roles nunca contienen valores secretos — solo referencias a variables del vault con el prefijo `vault_`.

### Crear el vault por primera vez

Si aún no existe `inventory/group_vars/all/vault.yml`, créalo así:

**1.** Crea el fichero con los secretos en texto plano:
```bash
cat > /tmp/vault.yml << 'EOF'
vault_win_password: "contraseña_windows"
vault_db_password: "contraseña_postgres"
EOF
```

**2.** Cífralo con Ansible Vault:
```bash
docker exec -it ansible ansible-vault encrypt /tmp/vault.yml --output inventory/group_vars/all/vault.yml
```

Introduce la contraseña del vault cuando se solicite. Guárdala también en `docker/ssh/.vault_pass` (ver sección [Contraseña del vault automática](#contraseña-del-vault-automática)).

---

### Variables actuales del vault

| Variable | Usada por | Descripción |
|---|---|---|
| `vault_win_password` | `group_vars/windows.yml` | Contraseña del usuario de gestión en hosts Windows |
| `vault_db_password` | `docker_container` (postgres) | Contraseña de PostgreSQL |
| `vault_mysql_root` | `mariadb` | Contraseña de root de MariaDB |
| `vault_user_hash` | `user_create` | Hash de contraseña de usuario del sistema |
| `vault_backup_pass` | `backup_remote` | Contraseña del recurso UNC de backup (Windows) |
| `vault_proxmox_token_secret` | `proxmox_snapshot` | Secreto del API token de Proxmox |

### Añadir un nuevo secreto

Cuando un nuevo rol necesite una contraseña, el flujo es siempre el mismo:

**1.** Descifra el vault:
```bash
docker exec -it ansible ansible-vault decrypt inventory/group_vars/all/vault.yml
```

**2.** Edita `inventory/group_vars/all/vault.yml` y añade la nueva variable:
```yaml
vault_db_password: mi_password
vault_nuevo_secreto: "valor"     # ← nueva entrada
```

**3.** Vuelve a cifrarlo:
```bash
docker exec -it ansible ansible-vault encrypt inventory/group_vars/all/vault.yml
```

**4.** En el rol o en `group_vars`, referencia la variable:
```yaml
alguna_password: "{{ vault_nuevo_secreto }}"
```

### Contraseña del vault automática

Como `vault.yml` está en `group_vars/all/`, Ansible intenta cargarlo en **cada ejecución**, incluso en playbooks que no usan secretos. Para no tener que escribir `--ask-vault-pass` cada vez, configura un fichero de contraseña:

**1.** Crea el fichero con tu contraseña del vault en la carpeta `ssh/` del proyecto Docker (se monta en el contenedor y persiste entre reinicios):
```bash
echo "tu_contraseña_vault" > docker/ssh/.vault_pass
chmod 600 docker/ssh/.vault_pass
```

**2.** Apunta a él en `ansible.cfg` (ya configurado):
```ini
vault_password_file = /root/.ssh/.vault_pass
```

A partir de ahí Ansible descifra el vault automáticamente y no necesitas añadir ningún flag extra a los comandos.

---

## Convenciones

- **`defaults/main.yml`** — variables con menor precedencia, siempre sobreescribibles.
- **`vars/main.yml`** — variables internas del rol, no pensadas para sobreescribir.
- **`group_vars/`** — variables compartidas por grupo de hosts.
- **Secretos** — usar Ansible Vault. Nunca subir contraseñas en texto plano. Los ficheros `vault*`, `secrets.yml` y similares están en `.gitignore`.
- **Colecciones** — los módulos `community.docker` requieren que Docker esté accesible en el host controlado.
