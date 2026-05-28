3\. Diseño e Implementación de la Base de Datos Corporativa  
Para que Innovate Tech pueda funcionar de manera ordenada y segura, necesitábamos una base de datos centralizada que actuara como columna vertebral de todas las aplicaciones corporativas. Este sistema se encarga de tres cosas principales: saber quién trabaja aquí y en qué departamento, controlar las comunicaciones por vídeo, y dejar rastro de todo lo que ocurre para poder auditarlo.

3.1 Arquitectura Física y Segregación de Servicios (AWS)  
Una de las primeras decisiones de diseño fue separar el servidor web del servidor de datos en dos máquinas distintas dentro de AWS. No es capricho — es sentido común de seguridad.

Servidor Web y Streaming (EC2-5): aquí vive Jitsi Meet, accesible desde el exterior a través de la IP pública 34.196.1.142.  
Servidor de Base de Datos (EC2-6): aquí corre MariaDB, con IP interna 100.48.147.77 y sin ninguna IP pública asignada.

Que el servidor de base de datos no tenga IP pública es intencionado: nadie desde Internet puede ni intentar conectarse a él directamente. Solo existe para la red interna.  
3.1.1 Configuración del Acceso de Red a MariaDB  
MariaDB, por defecto, solo escucha conexiones locales. Como el servidor web está en otra máquina, hubo que ajustar dos cosas:

En el fichero de configuración /etc/mysql/mariadb.conf.d/50-server.cnf, se cambió la directiva para que escuche en todas las interfaces:

ini    bind-address \= 0.0.0.0

En el Security Group de AWS, se abrió el puerto TCP 3306, pero solo para conexiones que vengan del servidor web (34.196.1.142). Cualquier otra IP lo tiene cerrado.

3.2 Diseño de la Base de Datos  
El modelo de datos se organizó en torno a dos preguntas concretas: ¿quién forma la empresa? y ¿cómo se comunica internamente?  
3.2.1 Gestión del Personal y Estructura Organizativa  
La estructura interna de la empresa se refleja en dos tablas que se relacionan entre sí:

Departamentos: cada área tiene un código único, un nombre y un teléfono de contacto.  
Empleados: identificados por su DNI, con sus datos básicos (nombre, apellidos, dirección, teléfono) y siempre vinculados a un departamento. Un empleado pertenece a un único departamento; un departamento puede tener muchos empleados.

3.2.2 Sistema de Comunicación Interna (Videollamadas y Streaming)  
Esta parte del modelo da soporte a todo lo que ocurre en Jitsi Meet:

Usuarios del Sistema: no todos los empleados tienen necesariamente cuenta en el sistema de comunicación. Esta entidad gestiona el correo corporativo, la extensión telefónica y si la cuenta está activa o bloqueada.  
Videollamadas / Llamadas: registra cada sesión de comunicación — cuánto duró, el enlace generado y las métricas básicas de uso.  
Configuración de Calidad: una tabla sencilla con perfiles predefinidos (alta, media, baja calidad) que se aplican dinámicamente según el ancho de banda disponible del usuario.

3.2.3 Modelo Relacional Resultante  
El esquema final, con sus claves primarias y foráneas, queda así:

DEPARTAMENTOS (codi\_dept \[PK\], nom\_dept, telefon\_dept)  
EMPLEADOS (dni \[PK\], nom, cognoms, adreça, telefon, codi\_dept \[FK\])  
USUARIOS\_SISTEMA (id\_usuari \[PK\], nom\_complet, correu\_electronic, extensio, estat, dni\_empleat \[FK\])  
CONFIG\_CALIDAD (id\_perfil \[PK\], descripcio\_qualitat, limitacio\_amplada\_banda)  
LLAMADAS (id\_trucada \[PK\], enllac\_videotrucada, data\_inici, durada\_minuts, id\_usuari \[FK\], id\_perfil \[FK\])  
AVISOS (id\_log \[PK\], usuari\_host, taula\_afectada, operacio\_intentada, data\_registre)

3.3 Gestión de Usuarios, Roles y Permisos  
La aplicación web nunca toca la cuenta root de MariaDB. Eso sería como darle las llaves de toda la casa a cualquiera que llame a la puerta. En su lugar, se definió un sistema de roles con permisos ajustados a lo que cada perfil realmente necesita hacer.  
3.3.1 Definición de Roles y Privilegios (DCL)  
Se crearon tres roles diferenciados:

Rol admin: acceso completo — puede leer, escribir, modificar estructuras y consultar la tabla de logs de auditoría.  
Rol vendes: permisos operativos básicos (SELECT, INSERT, UPDATE) sobre clientes, pedidos y metadatos de llamadas. No puede tocar la estructura de la base de datos.  
Rol administracio: gestiona únicamente lo relacionado con personal: empleados y departamentos.

Un ejemplo de cómo se configura esto en SQL:  
sql-- Creación de roles  
CREATE ROLE IF NOT EXISTS 'admin\_role', 'vendes\_role';

\-- Permisos del rol operativo  
GRANT SELECT, INSERT, UPDATE ON Innovatetech.clients TO 'vendes\_role';  
GRANT SELECT, INSERT, UPDATE ON Innovatetech.trucades TO 'vendes\_role';

\-- Usuario de aplicación vinculado al rol  
CREATE USER 'usr\_vendes\_01'@'100.48.147.77' IDENTIFIED BY 'PasswordSegura2026\*';  
GRANT 'vendes\_role' TO 'usr\_vendes\_01'@'100.48.147.77';  
SET DEFAULT ROLE 'vendes\_role' FOR 'usr\_vendes\_01'@'100.48.147.77';  
FLUSH PRIVILEGES;  
3.3.2 Mecanismos de Auditoría Automatizada (Triggers)  
Para no depender de que nadie "se olvide" de registrar algo, la auditoría está automatizada directamente en el motor mediante triggers:

Control de cuota de comunicación: antes de insertar cualquier llamada, un trigger comprueba si el usuario ha agotado sus minutos mensuales. Si los ha superado, lanza un SIGNAL SQLSTATE y la inserción se cancela automáticamente.  
Log de intentos no autorizados: si un usuario con permisos limitados intenta hacer algo que no debería, el sistema lo registra solo en la tabla AVISOS — quién lo intentó, qué operación era y en qué momento exacto — sin que el usuario infractor pueda evitarlo ni saberlo.