# ansible-lab-01

Proyecto de Ansible que administra un servidor Ubuntu 24.04 en Google Cloud desde la laptop (control node).

## Estructura

| Archivo | Descripcion |
| --- | --- |
| inventory.ini | Define el grupo webserver y como conectarse por SSH |
| playbook.yml | Instala y configura Nginx, dig y Node.js |
| .gitignore | Evita subir llaves privadas al repositorio |

## Uso

Probar la conexion:

    ansible webserver -i inventory.ini -m ansible.builtin.ping

Validar y ejecutar el playbook:

    ansible-playbook -i inventory.ini playbook.yml --syntax-check
    ansible-playbook -i inventory.ini playbook.yml
