# ansible-lab-01

Proyecto de Ansible que administra un servidor Ubuntu 24.04 alojado en Google Cloud
(`us-central1-a`) desde la laptop, que actúa como **control node**.

```
Laptop (control node)  →  SSH  →  Servidor Ubuntu (grupo webserver)
                                   nginx · dig · node · docker
```

## Estructura

| Archivo | Descripción |
| --- | --- |
| `inventory.ini` | Define el grupo `webserver` y cómo conectarse por SSH |
| `playbook.yml` | Instala y configura Nginx, dig y Node.js |
| `playbook-docker.yml` | Reto adicional: instala Docker Engine |
| `.gitignore` | Evita subir llaves privadas al repositorio |

---

## 1. Instalación de Ansible en el control node

```bash
sudo apt update && sudo apt install ansible -y
ansible --version
```

Salida:

```
ansible [core 2.16.3]
  config file = None
  configured module search path = ['/home/metrica/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/lib/python3/dist-packages/ansible
  ansible collection location = /home/metrica/.ansible/collections:/usr/share/ansible/collections
  executable location = /bin/ansible
  python version = 3.12.3 (main, Aug 31 2026, 10:18:26) [GCC 13.3.0] (/usr/bin/python3)
  jinja version = 3.1.2
  libyaml = True
```

---

## 2. Inventory

Contenido de `inventory.ini`:

```ini
[webserver]
server1 ansible_host=34.27.206.248 ansible_user=arturo ansible_ssh_private_key_file=~/.ssh/id_ed25519_clase
```

Verificación de cómo Ansible interpreta el inventory:

```bash
ansible-inventory -i inventory.ini --graph
```

Salida:

```
@all:
  |--@ungrouped:
  |--@webserver:
  |  |--server1
```

---

## 3. Primera conexión con Ansible

```bash
ansible webserver -i inventory.ini -m ansible.builtin.ping
```

Salida:

```
server1 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
```

> `ansible.builtin.ping` no es un ping ICMP: Ansible entra por SSH, copia un módulo
> Python al servidor, lo ejecuta y recoge el resultado.

---

## 4. Comandos ad-hoc

```bash
ansible webserver -i inventory.ini -m ansible.builtin.command -a "date"
ansible webserver -i inventory.ini -m ansible.builtin.command -a "hostname"
ansible webserver -i inventory.ini -m ansible.builtin.command -a "uptime"
ansible webserver -i inventory.ini -m ansible.builtin.command -a "free -h"
ansible webserver -i inventory.ini -m ansible.builtin.command -a "df -h"
```

Salida:

```
server1 | CHANGED | rc=0 >>
Thu Sep 24 03:27:45 UTC 2026

server1 | CHANGED | rc=0 >>
lab-arturo.us-central1-a.c.project-1-380901.internal

server1 | CHANGED | rc=0 >>
 03:27:56 up 5 days, 27 min,  1 user,  load average: 0.00, 0.04, 0.01

server1 | CHANGED | rc=0 >>
               total        used        free      shared  buff/cache   available
Mem:           1.9Gi       483Mi       1.1Gi       1.1Mi       479Mi       1.4Gi
Swap:             0B          0B          0B

server1 | CHANGED | rc=0 >>
Filesystem      Size  Used Avail Use% Mounted on
/dev/root        14G  2.7G   11G  21% /
tmpfs           981M     0  981M   0% /dev/shm
tmpfs           393M  976K  392M   1% /run
tmpfs           5.0M     0  5.0M   0% /run/lock
efivarfs        256K   32K  220K  13% /sys/firmware/efi/efivars
/dev/sda16      881M   85M  734M  11% /boot
/dev/sda15      105M  6.2M   99M   6% /boot/efi
tmpfs           197M   12K  196M   1% /run/user/1001
```

> El módulo `command` siempre reporta `CHANGED` aunque solo lea información:
> no puede saber si el comando modificó algo. Los módulos especializados
> (`apt`, `service`) sí lo distinguen.

### Ad-hoc con `become` (instalar un paquete)

```bash
ansible webserver -i inventory.ini -b -m ansible.builtin.apt -a "name=htop state=present update_cache=yes"
```

Salida:

```
server1 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "cache_update_time": 1790220522,
    "cache_updated": true,
    "changed": false
}
```

```bash
ansible webserver -i inventory.ini -m ansible.builtin.command -a "htop --version"
```

```
server1 | CHANGED | rc=0 >>
htop 3.3.0
```

> `changed: false` porque htop ya estaba instalado. `state: present` significa
> "asegúrate de que esté", no "instálalo".

---

## 5. Playbook: Nginx + dig + Node.js

Validación de sintaxis:

```bash
ansible-playbook -i inventory.ini playbook.yml --syntax-check
```

```
playbook: playbook.yml
```

Primera ejecución:

```bash
ansible-playbook -i inventory.ini playbook.yml
```

```
PLAY [Configure web server] ****************************************************

TASK [Gathering Facts] *********************************************************
ok: [server1]

TASK [Update apt cache] ********************************************************
changed: [server1]

TASK [Install Nginx] ***********************************************************
ok: [server1]

TASK [Install dig] *************************************************************
ok: [server1]

TASK [Install Node.js] *********************************************************
changed: [server1]

TASK [Start and enable Nginx] **************************************************
ok: [server1]

PLAY RECAP *********************************************************************
server1                    : ok=6    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

---

## 6. Verificación del resultado

```bash
ansible webserver -i inventory.ini -m command -a "nginx -v"
ansible webserver -i inventory.ini -m command -a "dig -v"
ansible webserver -i inventory.ini -m command -a "node --version"
```

Salida:

```
server1 | CHANGED | rc=0 >>
nginx version: nginx/1.24.0 (Ubuntu)

server1 | CHANGED | rc=0 >>
DiG 9.18.39-0ubuntu0.24.04.7-Ubuntu

server1 | CHANGED | rc=0 >>
v18.19.1
```

---

## 7. Idempotencia

Segunda ejecución del mismo playbook, sin cambiar nada:

```bash
ansible-playbook -i inventory.ini playbook.yml
```

```
PLAY [Configure web server] ****************************************************

TASK [Gathering Facts] *********************************************************
ok: [server1]

TASK [Update apt cache] ********************************************************
changed: [server1]

TASK [Install Nginx] ***********************************************************
ok: [server1]

TASK [Install dig] *************************************************************
ok: [server1]

TASK [Install Node.js] *********************************************************
ok: [server1]

TASK [Start and enable Nginx] **************************************************
ok: [server1]

PLAY RECAP *********************************************************************
server1                    : ok=6    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

`changed=2` → `changed=1`. Nginx, dnsutils y Node.js ya estaban instalados, así que
Ansible no volvió a tocarlos. El único `changed` que queda es `Update apt cache`,
porque actualizar la caché **es** una acción en sí misma: no tiene un estado
"ya está actualizada" que comprobar.

---

## 8. Reto adicional: Docker

`playbook-docker.yml` instala Docker Engine desde el **repositorio oficial de Docker**
(no el paquete `docker.io` de Ubuntu, que está desactualizado): prerequisitos →
llave GPG → repositorio → instalación → servicio → usuario en el grupo `docker`.

```bash
ansible-playbook -i inventory.ini playbook-docker.yml
```

Primera ejecución:

```
PLAY [Install Docker] **********************************************************

TASK [Gathering Facts] *********************************************************
ok: [server1]

TASK [Install prerequisites] ***************************************************
ok: [server1]

TASK [Create keyrings directory] ***********************************************
ok: [server1]

TASK [Download Docker GPG key] *************************************************
changed: [server1]

TASK [Add Docker repository] ***************************************************
changed: [server1]

TASK [Install Docker Engine] ***************************************************
changed: [server1]

TASK [Start and enable Docker] *************************************************
ok: [server1]

TASK [Add user to docker group] ************************************************
changed: [server1]

PLAY RECAP *********************************************************************
server1                    : ok=8    changed=4    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

### Verificación

```bash
ansible webserver -i inventory.ini -m command -a "docker --version"
```

```
server1 | CHANGED | rc=0 >>
Docker version 29.8.1, build 4a63305
```

```bash
ansible webserver -i inventory.ini -b -m command -a "docker run --rm hello-world"
```

```
server1 | CHANGED | rc=0 >>

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.
```

### Idempotencia

Segunda ejecución del playbook de Docker:

```
PLAY RECAP *********************************************************************
server1                    : ok=8    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

`changed=0`: el playbook puede ejecutarse nuevamente sin realizar cambios innecesarios,
que es lo que pedía el reto.

---

## 9. CI/CD con GitHub Actions

```
Repositorio Ansible  →  push a main  →  GitHub-hosted runner  →  SSH · puerto 22  →  Servidor destino
                                         ubuntu-latest                                cambios aplicados
                                         ansible-playbook
```

El workflow `.github/workflows/ansible.yml` tiene dos jobs encadenados:

| Job | Cuándo corre | Qué hace |
| --- | --- | --- |
| `Validar sintaxis` | Pull Request **y** push | `--syntax-check` de ambos playbooks. No toca el servidor. Es el **CI**. |
| `Aplicar en el servidor` | solo push a `main`, vía `if: github.event_name == 'push'` | Escribe la llave SSH desde el Secret y ejecuta los dos playbooks. Es el **CD**. |

Secrets configurados en el repositorio: `SSH_PRIVATE_KEY` y `SERVER_HOST`.

La ruta de la llave se sobreescribe con `-e ansible_ssh_private_key_file=$HOME/.ssh/id_ed25519`,
porque en el runner la llave no vive en la misma ruta que en la laptop. Las *extra vars* (`-e`)
tienen la precedencia más alta en Ansible, así el mismo `inventory.ini` sirve en los dos entornos
sin modificarlo.

### Evidencia: en el Pull Request solo corre el CI

```
All checks have passed — 1 skipped, 1 successful check

  ⊘  Ansible CI/CD / Aplicar en el servidor (pull_request)   Skipped
  ✓  Ansible CI/CD / Validar sintaxis (pull_request)         Successful in 45s
```

### Evidencia: al mergear a main corre el CD

Comando que ejecuta el runner:

```
Run ansible-playbook -i inventory.ini playbook.yml -e ansible_ssh_private_key_file=$HOME/.ssh/id_ed25519
```

Salida del primer playbook:

```
PLAY [Configure web server] ****************************************************

TASK [Gathering Facts] *********************************************************
ok: [server1]

TASK [Update apt cache] ********************************************************
changed: [server1]

TASK [Install Nginx] ***********************************************************
ok: [server1]

TASK [Install dig] *************************************************************
ok: [server1]

TASK [Install Node.js] *********************************************************
ok: [server1]

TASK [Start and enable Nginx] **************************************************
ok: [server1]

PLAY RECAP *********************************************************************
server1                    : ok=6    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

Salida del playbook de Docker:

```
PLAY [Install Docker] **********************************************************

TASK [Gathering Facts] *********************************************************
ok: [server1]

TASK [Install prerequisites] ***************************************************
ok: [server1]

TASK [Create keyrings directory] ***********************************************
ok: [server1]

TASK [Download Docker GPG key] *************************************************
ok: [server1]

TASK [Add Docker repository] ***************************************************
ok: [server1]

TASK [Install Docker Engine] ***************************************************
ok: [server1]

TASK [Start and enable Docker] *************************************************
ok: [server1]

TASK [Add user to docker group] ************************************************
ok: [server1]

PLAY RECAP *********************************************************************
server1                    : ok=8    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

`changed=0` en el segundo playbook: el pipeline aplicó la configuración y el servidor
ya estaba en el estado deseado. La idempotencia se cumple también desde CI/CD, no solo
al ejecutar Ansible desde la laptop.

---

## 10. Verificación directa en el servidor

Comprobación del estado final entrando por SSH a la VM, **sin pasar por Ansible**:
lo que sigue es lo que realmente quedó instalado y corriendo en la máquina.

```bash
ssh arturo@34.27.206.248
```

```bash
systemctl is-active nginx docker pythonapp
nginx -v; node --version; dig -v; docker --version; htop --version
id arturo
docker ps -a --format "table {{.Image}}\t{{.Status}}"
```

Salida:

```
=== Servicios ===
active
active
active

=== Versiones ===
nginx version: nginx/1.24.0 (Ubuntu)
v18.19.1
DiG 9.18.39-0ubuntu0.24.04.7-Ubuntu
Docker version 29.8.1, build 4a63305
htop 3.3.0

=== Grupos de arturo ===
uid=1001(arturo) gid=1002(arturo) groups=1002(arturo),4(adm),20(dialout),24(cdrom),
25(floppy),29(audio),30(dip),44(video),46(plugdev),105(lxd),111(netdev),1000(ubuntu),
1001(google-sudoers),988(docker)

=== Docker funcionando ===
IMAGE     STATUS
```

Lectura de la evidencia:

| Comprobación | Resultado | Qué demuestra |
| --- | --- | --- |
| `systemctl is-active nginx` | `active` | La tarea *Start and enable Nginx* del playbook surtió efecto |
| `systemctl is-active docker` | `active` | La tarea *Start and enable Docker* surtió efecto |
| `node --version` | `v18.19.1` | La tarea *Install Node.js* instaló el paquete |
| `dig -v` | `DiG 9.18.39` | La tarea *Install dig* instaló `dnsutils` |
| `htop 3.3.0` | instalado | Resultado del comando **ad-hoc**, no del playbook |
| `id arturo` incluye `988(docker)` | ✅ | La tarea *Add user to docker group* funcionó: `arturo` puede usar Docker sin `sudo` |
| `docker ps -a` vacío | ✅ | Correcto: el contenedor de prueba se ejecutó con `--rm`, así que se borró al terminar |

> `pythonapp` también aparece como `active`, pero ese servicio no lo gestiona este
> proyecto: pertenece al repositorio `python-web-app` y comparte el mismo servidor.

---

## Conceptos

| Concepto | Qué es, en este proyecto |
| --- | --- |
| **Control Node** | La máquina desde donde se ejecuta Ansible: la laptop. Ansible solo se instala aquí, no en el servidor. |
| **Inventory** | El archivo que declara qué máquinas administrar y cómo conectarse: `inventory.ini`. |
| **Grupo de hosts** | Una etiqueta que agrupa servidores: `[webserver]`. Un solo comando configura todos los hosts del grupo. |
| **Módulo** | La unidad de trabajo: `apt`, `service`, `file`, `user`, `get_url`. Cada uno sabe cómo llevar una cosa al estado pedido. |
| **Comando ad-hoc** | Una acción puntual desde la terminal, sin archivo. Útil para diagnóstico o cambios rápidos. |
| **Playbook** | Archivo YAML versionado con la configuración completa. Reproducible y auditable, a diferencia del ad-hoc. |
| **become** | Ejecutar con privilegios elevados (sudo). Sin él, `apt` falla por permisos. |
| **Idempotencia** | Ejecutar N veces deja el mismo resultado que ejecutar una. Demostrado arriba: `changed=4` → `changed=0`. |

---

## Uso

```bash
# Probar la conexión
ansible webserver -i inventory.ini -m ansible.builtin.ping

# Validar y ejecutar el playbook principal
ansible-playbook -i inventory.ini playbook.yml --syntax-check
ansible-playbook -i inventory.ini playbook.yml

# Reto adicional: Docker
ansible-playbook -i inventory.ini playbook-docker.yml
```
