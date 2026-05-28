# EC2-1 - Servidor Web (Apache) + SFTP

## Descripció
Aquest servidor allotja el servei web d'InnovateTech mitjançant Apache2, i el servei de transferència segura de fitxers (SFTP) autenticat amb els usuaris del directori actiu LDAP (EC2-2).

- **IP Pública**: 3.227.195.62
- **IP Privada**: 172.31.30.217
- **Sistema Operatiu**: Ubuntu Server 24.04 LTS
- **Tipus d'instància**: t2.micro

---

## Instal·lació i Configuració

### 1. Creació de la instància EC2
- AMI: Ubuntu Server 24.04 LTS
- Tipus: t2.micro (free tier)
- Par de claus: `PROYECTO_TRANSVERSAL.pem`
- Security Group `sg-web-sftp` amb les següents regles d'entrada:
  - SSH: port 22, source 0.0.0.0/0
  - HTTP: port 80, source 0.0.0.0/0
  - HTTPS: port 443, source 0.0.0.0/0

### 2. Connexió inicial i creació d'usuari administrador
Connexió inicial amb l'usuari per defecte:
```bash
ssh -i PROYECTO_TRANSVERSAL.pem ubuntu@3.227.195.62
```

Creació de l'usuari administrador específic `adminitb` (no s'utilitza l'usuari per defecte):
```bash
sudo adduser adminitb
sudo usermod -aG sudo adminitb
sudo mkdir /home/adminitb/.ssh
sudo cp /home/ubuntu/.ssh/authorized_keys /home/adminitb/.ssh/
sudo chown -R adminitb:adminitb /home/adminitb/.ssh
sudo chmod 700 /home/adminitb/.ssh
sudo chmod 600 /home/adminitb/.ssh/authorized_keys
```

A partir d'aquí totes les connexions es fan amb l'usuari `adminitb`:
```bash
ssh -i PROYECTO_TRANSVERSAL.pem adminitb@3.227.195.62
```

### 3. Instal·lació d'Apache
```bash
sudo apt update
sudo apt install apache2 -y
sudo systemctl enable apache2
sudo systemctl start apache2
```

### 4. Configuració del SFTP
OpenSSH ja ve instal·lat a Ubuntu. Es configura per autenticar els usuaris del grup `sftpusers`:
```bash
sudo groupadd sftpusers
sudo nano /etc/ssh/sshd_config
```

S'afegeix al final del fitxer:
```
Subsystem sftp internal-sftp
Match Group sftpusers
    ChrootDirectory /home/%u
    ForceCommand internal-sftp
    AllowTcpForwarding no
```

```bash
sudo systemctl restart ssh
```

### 5. Connexió amb LDAP (EC2-2)
Instal·lació dels mòduls necessaris per autenticar usuaris via LDAP:
```bash
sudo apt install libpam-ldap libnss-ldap ldap-utils nslcd -y
```

Durant la instal·lació es configura:
- URI del servidor LDAP: `ldap://172.31.28.178`
- Distinguished name: `dc=innovatetech,dc=local`

Edició de `/etc/nsswitch.conf` per incloure LDAP:
```
passwd:     files ldap
group:      files ldap
shadow:     files ldap
```

Edició de `/etc/nslcd.conf`:
```
uri ldap://172.31.28.178
base dc=innovatetech,dc=local
```

Reinici del servei:
```bash
sudo systemctl restart nslcd
```

---

## Proves de Funcionament

### Apache
Accés via navegador a `http://3.227.195.62` mostra la pàgina per defecte d'Apache2.

Verificació de l'estat del servei:
```bash
sudo systemctl status apache2
```

### LDAP
Verificació que EC2-1 veu els usuaris del LDAP d'EC2-2:
```bash
getent passwd juancarlos
```

Verificació de connectivitat amb EC2-2:
```bash
telnet 172.31.28.178 389
```

---

## Incidències i Solucions

### Problema 1: IP incorrecta en la configuració de nslcd
Durant la configuració de la connexió amb LDAP, es va introduir la IP `172.31.28.170` en lloc de la IP correcta `172.31.28.178`. Això va causar que `getent passwd juancarlos` no retornés res.

**Solució**: Es va reconfigurar `ldap-auth-config` amb la IP correcta:
```bash
sudo dpkg-reconfigure ldap-auth-config
```

### Problema 2: Ansible no podia connectar a EC2-2
Ansible donava error `Permission denied (publickey)` en intentar connectar a EC2-2 perquè l'usuari `adminitb` d'EC2-2 no tenia la carpeta `.ssh` ni el fitxer `authorized_keys`.

**Solució**: Es va crear la carpeta i copiar la clau pública des del usuari ubuntu:
```bash
sudo mkdir -p /home/adminitb/.ssh
sudo cp /home/ubuntu/.ssh/authorized_keys /home/adminitb/.ssh/authorized_keys
sudo chown -R adminitb:adminitb /home/adminitb/.ssh
sudo chmod 700 /home/adminitb/.ssh
sudo chmod 600 /home/adminitb/.ssh/authorized_keys
```

### Problema 3: Ansible requeria contrasenya de sudo
Ansible fallava amb `sudo: a password is required` perquè l'usuari `adminitb` necessitava contrasenya per executar comandes amb sudo.

**Solució**: Es va afegir la regla NOPASSWD al fitxer sudoers:
```bash
sudo visudo
# S'afegeix: adminitb ALL=(ALL) NOPASSWD: ALL
```

---

## Justificació de Tecnologies

- **Apache2**: Servidor web de codi obert, ampliament utilitzat en entorns empresarials, lleuger i fàcil de configurar.
- **OpenSSH SFTP**: Integrat al sistema operatiu, segur i compatible amb autenticació LDAP.
- **nslcd + libpam-ldap**: Permeten integrar l'autenticació de Linux amb un servidor LDAP extern de forma transparent.
