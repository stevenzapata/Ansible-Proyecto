# Ansible-Proyecto

Proyecto Ansible con roles y playbooks reutilizables para administración de servidores Linux.

## Estructura

```
Ansible-Proyecto/
├── ansible.cfg              # Configuración global de Ansible
├── inventory/
│   ├── hosts                # Inventario de hosts
│   └── group_vars/
│       └── all.yml          # Variables para todos los grupos
├── group_vars/
│   └── all.yml              # Variables globales extra
├── host_vars/               # Variables por host individual
├── playbooks/
│   ├── site.yml             # Playbook maestro
│   ├── update_system.yml    # Actualizar el sistema
│   ├── setup_ssh.yml        # Instalar SSH y copiar credenciales
│   ├── setup_nginx.yml      # Instalar servidor web Nginx
│   └── setup_apache.yml     # Instalar servidor web Apache
└── roles/
    ├── common/              # Paquetes base y configuración general
    ├── update/              # Actualización del sistema operativo
    ├── ssh/                 # Instalación y hardening de SSH
    ├── nginx/               # Servidor web Nginx
    └── apache/              # Servidor web Apache
```

## Requisitos

- Ansible >= 2.12
- Python >= 3.8
- Colecciones: `ansible.posix`, `community.general`

```bash
ansible-galaxy collection install ansible.posix community.general
```

## Uso

### Editar el inventario

Ajusta `inventory/hosts` con las IPs de tus servidores:

```ini
[webservers]
web01 ansible_host=192.168.1.10
```

### Ejecutar playbooks individuales

```bash
# Actualizar sistema
ansible-playbook playbooks/update_system.yml

# Instalar y configurar SSH
ansible-playbook playbooks/setup_ssh.yml

# Instalar Nginx
ansible-playbook playbooks/setup_nginx.yml

# Instalar Apache
ansible-playbook playbooks/setup_apache.yml

# Todo de una vez (playbook maestro)
ansible-playbook playbooks/site.yml
```

### Variables frecuentes por línea de comandos

```bash
# Cambiar hosts objetivo
ansible-playbook playbooks/setup_nginx.yml -e "target_hosts=web01"

# Pasar nombre de dominio y puerto
ansible-playbook playbooks/setup_nginx.yml -e "server_name=ejemplo.com http_port=8080"

# Permitir reinicio automático tras actualización
ansible-playbook playbooks/update_system.yml -e "update_allow_reboot=true"
```

### Copiar clave SSH a un usuario

En `playbooks/setup_ssh.yml` o como variable extra:

```bash
ansible-playbook playbooks/setup_ssh.yml -e '{
  "ssh_users": [{"name": "deployer", "public_key": "ssh-rsa AAAA..."}]
}'
```

## Convenciones

- Los **defaults** de cada rol van en `roles/<rol>/defaults/main.yml` (menor precedencia).
- Las **variables de grupo** van en `inventory/group_vars/` o `group_vars/`.
- Secretos y vaults **nunca** se suben al repositorio (ver `.gitignore`).
