# EC2-2 - Directori Actiu (OpenLDAP)

## Descripció
Aquest servidor allotja el servei de directori actiu d'InnovateTech mitjançant OpenLDAP. Centralitza la gestió d'usuaris de l'empresa, permetent que altres serveis (com el SFTP d'EC2-1) s'autentiquin contra aquest directori.

- **IP Pública**: 98.83.200.85
- **IP Privada**: 172.31.28.178
- **Sistema Operatiu**: Ubuntu Server 24.04 LTS
- **Tipus d'instància**: t2.micro

---

## Instal·lació i Configuració

### 1. Creació de la instància EC2
- AMI: Ubuntu Server 24.04 LTS
- Tipus: t2.micro (free tier)
- Par de claus: `PROYECTO_TRANSVERSAL.pem`
- Security Group `sg-ldap` amb les següents regles d'entrada:
  - SSH: port 22, source 0.0.0.0/0
  - LDAP: port 389, source 0.0.0.0/0
<img width="902" height="762" alt="image" src="https://github.com/user-attachments/assets/8d4dd63d-cbeb-4c99-809f-af10a6ed1849" />

### 2. Connexió inicial i creació d'usuari administrador
Connexió inicial amb l'usuari per defecte:
```bash
ssh -i PROYECTO_TRANSVERSAL.pem ubuntu@98.83.200.85
```

Creació de l'usuari administrador específic `adminitb`:
```bash
sudo adduser adminitb
sudo usermod -aG sudo adminitb
sudo mkdir -p /home/adminitb/.ssh
sudo cp /home/ubuntu/.ssh/authorized_keys /home/adminitb/.ssh/
sudo chown -R adminitb:adminitb /home/adminitb/.ssh
sudo chmod 700 /home/adminitb/.ssh
sudo chmod 600 /home/adminitb/.ssh/authorized_keys
```

### 3. Instal·lació d'OpenLDAP
```bash
sudo apt update
sudo apt install slapd ldap-utils -y
```

### 4. Configuració d'OpenLDAP
```bash
sudo dpkg-reconfigure slapd
```
<img width="897" height="495" alt="image" src="https://github.com/user-attachments/assets/818a6818-2e06-4eb9-9727-0e1c073d0131" />
<img width="901" height="645" alt="image" src="https://github.com/user-attachments/assets/a5e88cfc-ae33-48dd-99c5-a2cb12ad33c6" />


Opcions seleccionades:
- Omit OpenLDAP server configuration? → **No**
- DNS domain name → `innovatetech.local`
- Organization name → `InnovateTech`
- Admin password → *@ITB2026*
- Remove database when purging? → **No**
- Move old database? → **Yes**

### 5. Creació de l'estructura de directoris
Fitxer `base.ldif`:
<img width="547" height="183" alt="image" src="https://github.com/user-attachments/assets/ac8cbf70-8a6e-4bab-879d-dcad371dd399" />

<img width="899" height="141" alt="image" src="https://github.com/user-attachments/assets/098d8460-eb38-4046-8554-ba6711856101" />


### 6. Creació d'usuari de prova
Per generar el hash de la contrasenya:
<img width="576" height="159" alt="image" src="https://github.com/user-attachments/assets/62d7baf8-e0b5-44a6-934d-68dea35859d5" />

Fitxer **user1.ldif:**

<img width="723" height="279" alt="image" src="https://github.com/user-attachments/assets/70d180a6-e716-4a78-910d-570634d4acba" />


```bash
ldapadd -x -D cn=admin,dc=innovatetech,dc=local -W -f user1.ldif
```

### 7. Configuració de sudoers per Ansible
```bash
sudo visudo
# S'afegeix: adminitb ALL=(ALL) NOPASSWD: ALL
```

---

## Proves de Funcionament

### Verificació de l'estructura LDAP
```bash
ldapsearch -x -LLL -H ldap:// -b dc=innovatetech,dc=local
```
<img width="895" height="189" alt="image" src="https://github.com/user-attachments/assets/133f59b5-516b-4e8d-9c7a-3c622c6f2081" />


### Verificació que EC2-1 veu els usuaris LDAP
Des d'EC2-1:
```bash
getent passwd juancarlos
```
<img width="645" height="57" alt="image" src="https://github.com/user-attachments/assets/d15c2422-a0dc-4194-b078-c5ca88aefffb" />

---

## Incidències i Solucions

### Problema 1: Usuari juancarlos no apareixia a EC2-1
Després de configurar la connexió LDAP a EC2-1, el comando `getent passwd juancarlos` no retornava res.

**Causa**: La IP configurada a `ldap-auth-config` i `nslcd.conf` era `172.31.28.170` en lloc de la correcta `172.31.28.178`.

**Solució**: Es va reconfigurar amb la IP correcta:
```bash
sudo dpkg-reconfigure ldap-auth-config
```

### Problema 2: L'usuari adminitb no tenia carpeta .ssh
Ansible no podia connectar a EC2-2 perquè l'usuari `adminitb` no tenia la carpeta `.ssh` ni el fitxer `authorized_keys`.

**Solució**:
```bash
sudo mkdir -p /home/adminitb/.ssh
sudo cp /home/ubuntu/.ssh/authorized_keys /home/adminitb/.ssh/authorized_keys
sudo chown -R adminitb:adminitb /home/adminitb/.ssh
sudo chmod 700 /home/adminitb/.ssh
sudo chmod 600 /home/adminitb/.ssh/authorized_keys
```

---

## Justificació de Tecnologies

- **OpenLDAP**: Implementació de codi obert del protocol LDAP, àmpliament utilitzada en entorns empresarials per a la gestió centralitzada d'usuaris. Lleuger, escalable i compatible amb la majoria de serveis Linux.
- **LDAP (port 389)**: Protocol estàndard per a la gestió de directoris. S'utilitza el port 389 (sense xifrat) per simplicitat en un entorn intern de proves.
