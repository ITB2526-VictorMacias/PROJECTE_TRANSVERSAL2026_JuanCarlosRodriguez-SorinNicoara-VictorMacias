# Ansible - Configuració automatitzada

## Descripció
Ansible s'utilitza per automatitzar la configuració de les màquines EC2-1 i EC2-2. Permet reproduir tota la configuració dels servidors de forma automàtica i documentada.

---

## Instal·lació

Ansible s'instal·la a **EC2-1**, que actua com a màquina de control:

```bash
sudo apt update
sudo apt install ansible -y
```

---

## Fitxers de configuració

### inventory.ini
Define les màquines que gestiona Ansible:

```ini
[websftp]
localhost ansible_connection=local

[ldap]
172.31.28.178 ansible_user=adminitb ansible_ssh_private_key_file=~/.ssh/PROYECTO_TRANSVERSAL.pem
```

### setup.yml
Playbook que automatitza la instal·lació i configuració de tots els serveis:

```yaml
---
- hosts: websftp
  become: yes
  tasks:
    - name: Instalar Apache
      apt:
        name: apache2
        state: present
    - name: Iniciar Apache
      service:
        name: apache2
        state: started
        enabled: yes
    - name: Instalar libpam-ldap y nslcd
      apt:
        name:
          - libpam-ldap
          - libnss-ldap
          - ldap-utils
          - nslcd
        state: present
    - name: Iniciar nslcd
      service:
        name: nslcd
        state: started
        enabled: yes

- hosts: ldap
  become: yes
  tasks:
    - name: Instalar OpenLDAP
      apt:
        name:
          - slapd
          - ldap-utils
        state: present
    - name: Iniciar slapd
      service:
        name: slapd
        state: started
        enabled: yes
```

---

## Execució

### Verificar connectivitat amb totes les màquines
```bash
ansible -i ~/inventory.ini all -m ping
```

Resultat esperat:
```
localhost | SUCCESS => { "ping": "pong" }
172.31.28.178 | SUCCESS => { "ping": "pong" }
```

### Executar el playbook
```bash
sudo ansible-playbook -i ~/inventory.ini ~/setup.yml
```

---

## Incidències i Solucions

### Problema 1: sudo: a password is required
Ansible fallava perquè l'usuari `adminitb` necessitava contrasenya per executar comandes amb sudo.

**Solució**: Afegir NOPASSWD al fitxer sudoers de les dues màquines:
```bash
sudo visudo
# S'afegeix: adminitb ALL=(ALL) NOPASSWD: ALL
```

### Problema 2: Permission denied (publickey) a EC2-2
Ansible no podia connectar a EC2-2 perquè el fitxer `.pem` estava a `/home/adminitb/.ssh/` però quan s'executava amb `sudo` el buscava a `/root/.ssh/`.

**Solució**: Copiar el fitxer `.pem` a la carpeta de root:
```bash
sudo cp ~/.ssh/PROYECTO_TRANSVERSAL.pem /root/.ssh/
sudo chmod 400 /root/.ssh/PROYECTO_TRANSVERSAL.pem
```

### Problema 3: Clau pública no trobada a EC2-2
L'usuari `adminitb` d'EC2-2 no tenia la carpeta `.ssh` ni el fitxer `authorized_keys`, impedint la connexió SSH.

**Solució**: Copiar la clau pública des del usuari ubuntu d'EC2-2:
```bash
sudo mkdir -p /home/adminitb/.ssh
sudo cp /home/ubuntu/.ssh/authorized_keys /home/adminitb/.ssh/authorized_keys
sudo chown -R adminitb:adminitb /home/adminitb/.ssh
sudo chmod 700 /home/adminitb/.ssh
sudo chmod 600 /home/adminitb/.ssh/authorized_keys
```

---

## Justificació de Tecnologies

- **Ansible**: Eina d'automatització de codi obert que permet configurar servidors de forma repetible i documentada mitjançant fitxers YAML. No requereix instal·lació d'agent als servidors gestionats, només SSH.
