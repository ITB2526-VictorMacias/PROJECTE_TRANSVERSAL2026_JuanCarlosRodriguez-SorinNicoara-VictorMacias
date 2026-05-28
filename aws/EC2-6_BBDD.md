\# 3\. Diseño e Implementación de la Base de Datos Corporativa

Para dar soporte operativo a la gestión organizativa, comunicativa y de seguridad de la empresa \*\*Innovate Tech\*\*, se ha diseñado e implementado una base de datos centralizada en un entorno relacional. Este sistema sirve como motor principal y fuente de datos para las aplicaciones corporativas, gestionando la estructura de personal, el control de las comunicaciones y la auditoría de seguridad.

\---

\#\# 3.1 Arquitectura Física y Segregación de Servicios (AWS)

Para garantizar el máximo rendimiento y aplicar políticas estrictas de seguridad perimetral, se ha separado completamente el entorno web del de datos en dos instancias EC2 diferenciadas dentro de la plataforma Amazon Web Services (AWS):

1\.  \*\*Servidor Web y Streaming (EC2-5):\*\* Aloja la plataforma de comunicación Jitsi Meet en la IP pública \`34.196.1.142\`.  
2\.  \*\*Servidor de Base de Dades (EC2-6):\*\* Aloja el motor relacional \*\*MariaDB\*\* en la IP privada interna \`100.48.147.77\`. 

Al no tener asociada ninguna IP pública, el servidor de base de datos queda completamente aislado y protegido de cualquier intento de ataque o escaneo directo desde Internet.

\#\#\# 3.1.1 Configuración del Acceso de Red a MariaDB  
Por defecto, MariaDB solo acepta conexiones locales (\`127.0.0.1\`). Para permitir que la aplicación web (EC2-5) se conecte de manera remota al motor de datos (EC2-6), se realizaron las siguientes configuraciones de red:

\* \*\*Modificación del fichero de configuración del servicio:\*\* Se accedió al fichero \`/etc/mysql/mariadb.conf.d/50-server.cnf\` y se modificó la directiva de red para permitir la escucha en todas las interfaces de red interna de la instancia:  
    \`\`\`ini  
    bind-address \= 0.0.0.0  
    \`\`\`  
\* \*\*Firewalling (AWS Security Groups):\*\* Se parametrizó el \*Security Group\* asociado a la instancia de Base de Datos (EC2-6) para abrir el port \*\*TCP 3306\*\*, restringiendo el origen de las conexiones exclusivamente a la IP o al grupo de seguridad del Servidor Web (\`34.196.1.142\`).

\---

\#\# 3.2 Diseño de la Base de Datos

A partir de los requerimientos funcionales de la empresa, el modelado de datos se ha estructurado en dos grandes bloques lógicos: la organización del personal interno y el control del sistema de comunicaciones multimedia.

\#\#\# 3.2.1 Gestión del Personal y Estructura Organizativa  
Se almacena la estructura corporativa mediante las entidades principales de empleados y departamentos:  
\* \*\*Departamentos:\*\* Identificados por un código único, almacenando el nombre oficial y el teléfono de contacto del área.  
\* \*\*Empleados:\*\* Identificados unívocamente por el DNI, registrando datos filiativos obligatorios (nombre, apellidos, dirección, teléfono). Cada empleado está adscrito de manera obligatoria a un único departamento (relación 1:N).

\#\#\# 3.2.2 Sistema de Comunicación Interna (Videollamadas y Streaming)  
Mapea el control de Jitsi Meet y los parámetros de calidad establecidos en la red:  
\* \*\*Usuarios del Sistema:\*\* Entidad vinculada a los empleados potenciales donde se gestiona el correo electrónico corporativo, la extensión telefónica asignada para las llamadas y su estado lógico operacional (activo/bloqueado).  
\* \*\*Videollamadas / Llamadas:\*\* Cada usuario tiene la capacidad de generar o participar en líneas de comunicación, registrando la duración, el enlace dinámico generado y las métricas básicas.  
\* \*\*Configuración de Calidad:\*\* Tabla parametrizada que almacena los perfiles de calidad de vídeo y audio (alta, media, baja) aplicados en función de las restricciones de ancho de banda que sufra el usuario.

\#\#\# 3.2.3 Modelo Relacional Resultante  
Se presenta la transformación lógica del diseño, definiendo exhaustivamente las Claves Primarias (\`PK\`) y Claves Foráneas (\`FK\`) que garantizan la integridad referencial del sistema informático:

\* \*\*DEPARTAMENTOS\*\* (\*\*codi\_dept\*\* \[PK\], nom\_dept, telefon\_dept)  
\* \*\*EMPLEADOS\*\* (\*\*dni\*\* \[PK\], nom, cognoms, adreça, telefon, \*codi\_dept\* \[FK\])  
\* \*\*USUARIOS\_SISTEMA\*\* (\*\*id\_usuari\*\* \[PK\], nom\_complet, correu\_electronic, extensio, estat, \*dni\_empleat\* \[FK\])  
\* \*\*CONFIG\_CALIDAD\*\* (\*\*id\_perfil\*\* \[PK\], descripcio\_qualitat, limitacio\_amplada\_banda)  
\* \*\*LLAMADAS\*\* (\*\*id\_trucada\*\* \[PK\], enllac\_videotrucada, data\_inici, durada\_minuts, \*id\_usuari\* \[FK\], \*id\_perfil\* \[FK\])  
\* \*\*AVISOS\*\* (\*\*id\_log\*\* \[PK\], usuari\_host, taula\_afectada, operacio\_intentada, data\_registre)

\---

\#\# 3.3 Gestión de Usuarios, Roles y Permisos

Para cumplir con los requerimientos de seguridad del proyecto, se ha implementado un control de acceso estricto basado en roles de base de datos (RBAC), evitando que la aplicación web utilice directamente la cuenta de administración global de la instancia (\`root\`).

\#\#\# 3.3.1 Definición de Roles y Privilegios (DCL)  
Mediante sentencias SQL se han definido roles con privilegios granulares adaptados a las necesidades de cada departamento:

1\.  \*\*Rol \`admin\`:\*\* Acceso global de lectura, escritura, modificación de estructuras de datos y auditoría absoluta de la tabla de logs (\`avisos\`).  
2\.  \*\*Rol \`vendes\`:\*\* Permisos DML reducidos (\`SELECT\`, \`INSERT\`, \`UPDATE\`) exclusivamente orientados a la interacción operativa de clientes, pedidos y metadatos de llamadas. Tiene prohibido de manera estricta realizar cambios estructurales (DDL).  
3\.  \*\*Rol \`administracio\`:\*\* Permisos DML sobre las tablas relacionadas con el personal corporativo (empleados y departamentos).

\*Ejemplo de script SQL de configuración de seguridad:\*  
\`\`\`sql  
\-- Creación de roles en la instancia  
CREATE ROLE IF NOT EXISTS 'admin\_role', 'vendes\_role';

\-- Asignación de privilegios al rol operativo  
GRANT SELECT, INSERT, UPDATE ON Innovatetech.clients TO 'vendes\_role';  
GRANT SELECT, INSERT, UPDATE ON Innovatetech.trucades TO 'vendes\_role';

\-- Creación de un usuario de aplicación asociado al rol  
CREATE USER 'usr\_vendes\_01'@'100.48.147.77' IDENTIFIED BY 'PasswordSegura2026\*';  
GRANT 'vendes\_role' TO 'usr\_vendes\_01'@'100.48.147.77';  
SET DEFAULT ROLE 'vendes\_role' FOR 'usr\_vendes\_01'@'100.48.147.77';  
FLUSH PRIVILEGES;

3.3.2 Mecanismos de Auditoría Automatizada (Triggers)  
Se han programado disparadores (triggers) directamente integrados dentro del motor de datos para automatizar auditorías de seguridad y controles de uso:

Control de Cuota de Comunicación: Un trigger se ejecuta en fase BEFORE INSERT en la tabla de llamadas, validando si el usuario ha superado la cuota de minutos mensuales asignada a su perfil. Si se supera el umbral, lanza un SIGNAL SQLSTATE que bloquea la inserción.

Tabla de Avisos y Logs de Auditoría: Diseñada para registrar cualquier intento no autorizado de alteración de datos. Un trigger audita de manera transparente los intentos de modificación hechos por usuarios con roles restringidos sobre tablas protegidas, insertando automáticamente en la tabla AVISOS el usuario host infractor, la operación intentada y la marca de tiempo exacta.

3.4 Estrategia de Administración del Motor: Consola CLI vs Entorno Gráfico  
3.4.1 Diagnóstico de Conflictos en el Despliegamiento de Adminer  
Inicialmente se planteó el despliegue de Adminer (interfaz gráfica basada en PHP) en la instancia web (EC2-5). No obstante, durante la fase de pruebas se detectaron incompatibilidades estructurales críticas:

Conflicto de Enrutamiento con Jitsi Meet: Jitsi Meet configura el servidor Nginx de una manera altamente restrictiva en la IP de producción (/etc/nginx/sites-available/34.196.1.142.conf). Cualquier petición hacia rutas o ficheros con extensión .php era interceptada automáticamente por el servicio de videoconferencia, interpretándola erróneamente como el nombre de una sala de llamada dinámica.

Colisión de Puertos de Red (Apache2 vs Nginx): Al instalar las dependencias de ejecución de PHP (apt install php), el gestor de paquetes de Ubuntu levantó de manera automática el servidor web Apache2 en segundo plano. Esto provocó un bloqueo inmediato por conflicto de sockets (Address already in use), ya que Nginx ya ocupaba el puerto 80 de manera exclusiva para dar servicio a Jitsi.

3.4.2 Decisión de Ingeniería: Administración Nativa por Línea de Comandos (CLI)  
Ante la incompatibilidad de los entornos en la máquina web y aplicando políticas de seguridad avanzadas (dones se desaconseja firmemente exponer paneles de administración de bases de datos a redes públicas para evitar ataques de fuerza bruta), el equipo de infraestructura tomó la decisión de descartar el entorno gráfico de Adminer.

Se estableció que todo el mantenimiento, verificación y auditoría de datos se realizará de manera 100% nativa y segura a través de la línea de comandos (CLI) conectando directamente a la instancia de base de datos (EC2-6):