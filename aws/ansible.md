# Ansible - Configuració automatitzada

## Descripció
Ansible s'utilitza per automatitzar la configuració de les màquines EC2-1 i EC2-2. Permet reproduir tota la configuració dels servidors de forma automàtica i documentada.

---

## Instal·lació

Ansible s'instal·la a **EC2-1**, que actua com a màquina de control:


-  sudo apt update
-  sudo apt install ansible -y


---

## Fitxers de configuració

### inventory.ini
Define les màquines que gestiona Ansible:

<img width="905" height="150" alt="image" src="https://github.com/user-attachments/assets/7fd51bad-5476-4248-884d-87ed1cbd2588" />


### setup.yml
Playbook que automatitza la instal·lació i configuració de tots els serveis:

<img width="821" height="719" alt="image" src="https://github.com/user-attachments/assets/62c2d7a9-bb50-47bb-ba9d-51e963b97d24" />

---

## Execució

### Verificar connectivitat amb totes les màquines
<img width="644" height="293" alt="image" src="https://github.com/user-attachments/assets/f8b0035d-b032-4c44-a255-12ad42b0a534" />

### Executar el playbook

<img width="898" height="655" alt="image" src="https://github.com/user-attachments/assets/28285401-9de0-4d16-8983-562ba609e4a7" />

---

## Incidències i Solucions

### Problema 1: sudo: a password is required
Ansible fallava perquè l'usuari adminitb necessitava contrasenya per executar comandes amb sudo.

**Solució**: Afegir NOPASSWD al fitxer sudoers de les dues màquines:

-  sudo visudo
# S'afegeix: adminitb ALL=(ALL) NOPASSWD: ALL


### Problema 2: Permission denied (publickey) a EC2-2
Ansible no podia connectar a EC2-2 perquè el fitxer .pem estava a /home/adminitb/.ssh/ però quan s'executava amb sudo el buscava a /root/.ssh/.

**Solució**: Copiar el fitxer .pem a la carpeta de root:

-  sudo cp ~/.ssh/PROYECTO_TRANSVERSAL.pem /root/.ssh/
-  sudo chmod 400 /root/.ssh/PROYECTO_TRANSVERSAL.pem


### Problema 3: Clau pública no trobada a EC2-2
L'usuari adminitb d'EC2-2 no tenia la carpeta .ssh ni el fitxer authorized_keys, impedint la connexió SSH.

**Solució**: Copiar la clau pública des del usuari ubuntu d'EC2-2:

sudo mkdir -p /home/adminitb/.ssh  
sudo cp /home/ubuntu/.ssh/authorized_keys /home/adminitb/.ssh/authorized_keys  
sudo chown -R adminitb:adminitb /home/adminitb/.ssh  
sudo chmod 700 /home/adminitb/.ssh  
sudo chmod 600 /home/adminitb/.ssh/authorized_keys  


---

## Justificació de Tecnologies

- **Ansible**: Eina d'automatització de codi obert que permet configurar servidors de forma repetible i documentada mitjançant fitxers YAML. No requereix instal·lació d'agent als servidors gestionats, només SSH.
