# Memoria de Configuración: Servidor Web y SFTP (EC2-1)

Este documento detalla los pasos seguidos para configurar el servidor principal de InnovateTech, que se encarga de la web corporativa y de permitir la subida de archivos mediante SFTP utilizando los usuarios alojados en el servidor LDAP (EC2-2).

## Datos de la Instancia
* IP Pública: 3.227.195.62
* IP Privada: 172.31.30.217
* Sistema Operativo: Ubuntu Server 24.04 LTS
* Tipo de instancia: t2.micro

---

## 1. Puesta en marcha y Seguridad
Lo primero ha sido lanzar la instancia en AWS con el Security Group sg-web-sftp. Hemos configurado las reglas de entrada para permitir tráfico por los puertos 22 (SSH), 80 (HTTP) y 443 (HTTPS).

Para mejorar la seguridad, no trabajaremos con el usuario ubuntu. He creado un usuario propio llamado adminitb y le he dado permisos de administrador:

**sudo adduser adminitb**

**sudo usermod -aG sudo adminitb**

Para poder entrar sin contraseña, he copiado la clave pública del usuario ubuntu al nuevo directorio .ssh de adminitb. Una vez hecho esto, el acceso se realiza mediante:
ssh -i PROYECTO_TRANSVERSAL.pem adminitb@3.227.195.62

---

## 2. Instalación del Servidor Web
He instalado Apache2 para servir la web de la empresa. El proceso ha sido sencillo: actualizar repositorios, instalar el paquete y habilitar el servicio para que arranque automáticamente si se reinicia la máquina.

**sudo apt update**

**sudo apt install apache2 -y**

**sudo systemctl enable apache2**

**sudo systemctl start apache2**

---

## 3. Configuración del servicio SFTP
El objetivo es que los usuarios puedan subir archivos de forma segura. He creado un grupo llamado sftpusers y he modificado el archivo de configuración de SSH para "enjaular" a estos usuarios en su carpeta personal.

Al final del fichero /etc/ssh/sshd_config he añadido lo siguiente:

Subsystem sftp internal-sftp
Match Group sftpusers
    ChrootDirectory /home/%u
    ForceCommand internal-sftp
    AllowTcpForwarding no

Después de esto, he reiniciado el servicio con sudo systemctl restart ssh.

---

## 4. Integración con el Directorio Activo (LDAP)
Para no tener que crear los usuarios localmente en esta máquina, la he conectado con el servidor LDAP (EC2-2). He instalado los paquetes necesarios para que el sistema reconozca usuarios externos:

**sudo apt install libpam-ldap libnss-ldap ldap-utils nslcd -y**

Durante la instalación he configurado la IP del servidor LDAP (172.31.28.178) y el nombre del dominio (dc=innovatetech,dc=local). 

Para que el sistema busque primero en los archivos locales y luego en LDAP, he editado /etc/nsswitch.conf añadiendo la opción ldap en las líneas de passwd, group y shadow.

---

## Pruebas de funcionamiento
Para verificar que todo está bien configurado, he realizado estas comprobaciones:

* Web: Al entrar en la IP pública desde un navegador, carga la página por defecto de Apache.
* LDAP: Al ejecutar getent passwd juancarlos, el sistema devuelve correctamente los datos del usuario que está en la otra máquina.
* Conectividad: He comprobado con telnet que hay conexión al puerto 389 de la IP privada del servidor LDAP.

---

## Incidencias y soluciones

1. Error en la IP del LDAP: Al configurar nslcd me equivoqué en un número de la IP privada. El comando getent no devolvía nada. Lo he solucionado reconfigurando el paquete con sudo dpkg-reconfigure ldap-auth-config.

2. Error de permisos en Ansible: Cuando intenté usar Ansible desde esta máquina hacia la EC2-2, daba fallo de clave pública. He tenido que crear manualmente la carpeta .ssh en el destino y copiar la clave autorizada con los permisos correctos (700 para la carpeta y 600 para el archivo).

3. Contraseña en sudo: Ansible se bloqueaba al pedir la contraseña de root. He editado el archivo sudoers con visudo para permitir que el usuario adminitb ejecute comandos sin que se le pida la clave (NOPASSWD).
