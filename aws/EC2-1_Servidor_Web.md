# EC2-1 - Servidor Web (Apache) + SFTP

## Descripció

Aquest servidor s'encarrega d'allotjar el servei web d'InnovateTech a través d'Apache2, i també el servei de transferència segura de fitxers (SFTP), que s'autentica amb els usuaris del directori actiu LDAP gestionat des de l'EC2-2.

- IP Pública: 3.227.195.62
- IP Privada: 172.31.30.217
- Sistema Operatiu: Ubuntu Server 24.04 LTS
- Tipus d'instància: t2.micro

---

## Instal·lació i Configuració

### 1. Creació de la instància EC2

S'ha creat la instància amb els paràmetres següents: AMI Ubuntu Server 24.04 LTS, tipus t2.micro (free tier) i el parell de claus "PROYECTO_TRANSVERSAL.pem". El Security Group creat, anomenat sg-web-sftp, té les regles d'entrada per SSH (port 22), HTTP (port 80) i HTTPS (port 443), totes amb source 0.0.0.0/0.

### 2. Connexió inicial i creació d'usuari administrador

La connexió inicial es fa amb l'usuari per defecte ubuntu:

    ssh -i PROYECTO_TRANSVERSAL.pem ubuntu@3.227.195.62

A continuació es crea l'usuari administrador específic "adminitb", ja que no es vol fer servir el compte per defecte. Es copia la clau pública i s'estableixen els permisos correctes a la carpeta .ssh:

    sudo adduser adminitb
    sudo usermod -aG sudo adminitb
    sudo mkdir /home/adminitb/.ssh
    sudo cp /home/ubuntu/.ssh/authorized_keys /home/adminitb/.ssh/
    sudo chown -R adminitb:adminitb /home/adminitb/.ssh
    sudo chmod 700 /home/adminitb/.ssh
    sudo chmod 600 /home/adminitb/.ssh/authorized_keys

A partir d'aquí totes les connexions es fan amb l'usuari adminitb:

    ssh -i PROYECTO_TRANSVERSAL.pem adminitb@3.227.195.62

### 3. Instal·lació d'Apache

S'instal·la Apache2 i s'activa perquè s'iniciï automàticament amb el sistema:

    sudo apt update
    sudo apt install apache2 -y
    sudo systemctl enable apache2
    sudo systemctl start apache2

### 4. Configuració del SFTP

OpenSSH ja ve instal·lat a Ubuntu per defecte. Es crea el grup "sftpusers" i es modifica el fitxer de configuració del servei SSH per forçar l'ús de l'SFTP intern i limitar cada usuari al seu directori home (chroot):

    sudo groupadd sftpusers
    sudo nano /etc/ssh/sshd_config

Al final del fitxer s'afegeix el bloc següent:

    Subsystem sftp internal-sftp
    Match Group sftpusers
        ChrootDirectory /home/%u
        ForceCommand internal-sftp
        AllowTcpForwarding no

Finalment es reinicia el servei:

    sudo systemctl restart ssh

### 5. Connexió amb LDAP (EC2-2)

Per poder autenticar els usuaris del directori actiu des d'aquest servidor, cal instal·lar els mòduls de PAM i NSS per LDAP:

    sudo apt install libpam-ldap libnss-ldap ldap-utils nslcd -y

Durant el procés d'instal·lació es configura la URI del servidor LDAP (ldap://172.31.28.178) i el Distinguished Name de base (dc=innovatetech,dc=local).

Després s'edita el fitxer /etc/nsswitch.conf per indicar al sistema que també consulti LDAP per resoldre usuaris, grups i contrasenyes:

    passwd:     files ldap
    group:      files ldap
    shadow:     files ldap

I el fitxer /etc/nslcd.conf amb les dades del servidor:

    uri ldap://172.31.28.178
    base dc=innovatetech,dc=local

Per aplicar els canvis:

    sudo systemctl restart nslcd

---

## Proves de Funcionament

### Apache

S'accedeix al navegador amb la IP pública http://3.227.195.62 i es comprova que apareix la pàgina de benvinguda per defecte d'Apache2. També es verifica l'estat del servei per confirmar que està actiu:

    sudo systemctl status apache2

### LDAP

Es comprova que el servidor EC2-1 és capaç de resoldre els usuaris definits a l'LDAP de l'EC2-2:

    getent passwd juancarlos

I la connectivitat de xarxa amb l'EC2-2 pel port 389:

    telnet 172.31.28.178 389

---

## Incidències i Solucions

### Problema 1: IP incorrecta en la configuració de nslcd

Durant la configuració de la connexió amb LDAP es va introduir la IP 172.31.28.170 en lloc de la IP correcta 172.31.28.178. Això va fer que la comanda "getent passwd juancarlos" no retornés cap resultat.

Solució: Es va tornar a executar la configuració de ldap-auth-config amb la IP correcta:

    sudo dpkg-reconfigure ldap-auth-config

### Problema 2: Ansible no podia connectar a EC2-2

Ansible retornava l'error "Permission denied (publickey)" en intentar accedir a l'EC2-2 perquè l'usuari adminitb d'aquella màquina no tenia creada la carpeta .ssh ni el fitxer authorized_keys.

Solució: Es va crear la carpeta manualment i es va copiar la clau pública des del compte ubuntu:

    sudo mkdir -p /home/adminitb/.ssh
    sudo cp /home/ubuntu/.ssh/authorized_keys /home/adminitb/.ssh/authorized_keys
    sudo chown -R adminitb:adminitb /home/adminitb/.ssh
    sudo chmod 700 /home/adminitb/.ssh
    sudo chmod 600 /home/adminitb/.ssh/authorized_keys

### Problema 3: Ansible requeria contrasenya de sudo

Ansible fallava amb el missatge "sudo: a password is required" perquè l'usuari adminitb necessitava introduir contrasenya per executar comandes amb sudo.

Solució: Es va afegir la regla NOPASSWD al fitxer sudoers perquè Ansible pogués executar comandes sense intervenció manual:

    sudo visudo
    # S'afegeix: adminitb ALL=(ALL) NOPASSWD: ALL

---

## Justificació de Tecnologies

S'ha escollit Apache2 com a servidor web perquè és una solució de codi obert molt estesa en entorns empresarials, lleuger i fàcil de configurar per a un projecte d'aquesta mida. El servei SFTP s'implementa directament amb OpenSSH, ja integrat al sistema operatiu, el que evita instal·lar programari addicional i garanteix compatibilitat amb l'autenticació LDAP. Finalment, nslcd i libpam-ldap permeten integrar l'autenticació de Linux amb el servidor LDAP extern de manera transparent per als usuaris.
