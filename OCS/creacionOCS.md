# Instalación de OCS Inventory NG 2.12.1 en Ubuntu

> **Proyecto:** TFG de FP — Sistema de inventario automatizado (Anexo III, apartado 11.4).  
> **Entorno:** máquina virtual Ubuntu con Apache ya instalado para Moodle, MariaDB y OCS Inventory NG 2.12.1.  
> **IP observada durante las pruebas:** `192.168.0.103` (puede cambiar; en un borrador del TFG figuraba `192.168.0.104`).  
> **Estado del proyecto:** consola web instalada y accesible; **inventario de agentes todavía pendiente** (error HTTP 500 y 0 máquinas).

## 1. Objetivo

Instalar OCS Inventory NG para centralizar información de hardware y software de los equipos. El sistema utiliza MariaDB para almacenar los datos, Apache/PHP para la consola de administración y Apache/mod_perl para recibir los inventarios enviados por los agentes.

En esta práctica se instala todo en el mismo servidor Ubuntu. **No es necesario reinstalar Apache**, porque ya lo utilizamos para Moodle.

## 2. Preparación del sistema

Conectarse por SSH desde Windows (sustituir la IP si cambia):

```powershell
ssh vboxuser@192.168.0.103
```

Comprobar el sistema y Apache:

```bash
lsb_release -a
ip a
sudo systemctl status apache2 --no-pager
```

Actualizar los índices de paquetes:

```bash
sudo apt update
```

**Explicación:** `lsb_release -a` identifica Ubuntu; `ip a` muestra la IP; `systemctl` comprueba el servidor web; `apt update` actualiza la lista de programas disponibles. No desinstalar ni modificar Moodle.

## 3. Instalar MariaDB y los módulos de PHP

```bash
sudo apt install -y \
  mariadb-server mariadb-client \
  php-mysql php-mbstring php-xml php-curl php-gd php-zip php-soap \
  libapache2-mod-php

sudo systemctl enable --now mariadb
sudo systemctl status mariadb --no-pager
```

**Explicación:** MariaDB almacena el inventario; `php-mysql` permite conectar la consola a la base de datos; `php-mbstring`, `php-xml`, `php-curl`, `php-gd`, `php-zip` y `php-soap` aportan funcionalidades utilizadas por OCS. `libapache2-mod-php` integra PHP con Apache.

## 4. Crear la base de datos de OCS Inventory

Abrir MariaDB como administrador:

```bash
sudo mariadb
```

Ejecutar estas instrucciones **dentro de MariaDB** en una instalación nueva:

```sql
CREATE DATABASE ocsweb;
CREATE USER 'ocsuid'@'localhost' IDENTIFIED BY 'CAMBIAR_POR_UNA_CONTRASENA_SEGURA';
GRANT ALL PRIVILEGES ON ocsweb.* TO 'ocsuid'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

**Explicación:**

- `CREATE DATABASE` crea `ocsweb`, donde se guardarán los inventarios.
- `CREATE USER` crea `ocsuid`, usuario que usará OCS para consultar y escribir en MariaDB.
- `GRANT ALL PRIVILEGES` le concede permisos sobre **esa base de datos**, no sobre todas.
- `FLUSH PRIVILEGES` actualiza los privilegios; `EXIT` sale de MariaDB.

**Seguridad:** la contraseña mostrada es un marcador, **no una contraseña real**. No publicar la contraseña utilizada en el servidor en GitHub. Si `ocsweb` y `ocsuid` ya existen, **no vuelvas a ejecutar los comandos de creación**.

Verificar la conexión sin exponer la contraseña en la consola:

```bash
mariadb -u ocsuid -p -h localhost ocsweb
```

Introducir la contraseña cuando la pida, y salir con `EXIT;`.

## 5. Instalar las dependencias Perl

```bash
sudo apt install -y \
  libapache2-mod-perl2 libapache-dbi-perl libarchive-zip-perl \
  libxml-simple-perl libnet-ip-perl libsoap-lite-perl \
  libdbi-perl libdbd-mysql-perl libswitch-perl \
  libmojolicious-perl libplack-perl libyaml-perl \
  make build-essential wget tar
```

**Explicación:** el servidor de comunicaciones de OCS está escrito en Perl y funciona mediante `mod_perl`. Las bibliotecas `DBI` y `DBD::mysql` permiten consultar MariaDB. `XML::Simple`, `Net::IP` y `Archive::Zip` procesan información de los inventarios. `Switch` fue especialmente importante: su ausencia provocó que Apache dejase de funcionar al activar OCS.

Comprobar el módulo que faltaba:

```bash
perl -MSwitch -e 'print "SWITCH OK\n"'
```

**Nota:** si los registros indican que falta `XML::Entities`, consultar el apartado de incidencias; ese módulo puede requerir instalación mediante CPAN según la versión de Ubuntu.

## 6. Descargar e instalar OCS Inventory NG 2.12.1

```bash
cd /tmp
wget -O OCSNG_UNIX_SERVER-2.12.1.tar.gz \
  https://github.com/OCSInventory-NG/OCSInventory-ocsreports/releases/download/2.12.1/OCSNG_UNIX_SERVER-2.12.1.tar.gz

tar -xzf OCSNG_UNIX_SERVER-2.12.1.tar.gz
cd OCSNG_UNIX_SERVER-2.12.1
sudo bash ./setup.sh
```

**Explicación:** `wget` descarga el paquete oficial, `tar` lo descomprime y `setup.sh` instala el servidor de comunicaciones y la interfaz de administración. Aquí usamos `bash` porque en nuestro entorno el instalador había mostrado el error `[[: not found` al invocarse con el intérprete `sh`.

En el asistente:

1. Confirmar la instalación del **Communication server** y de la **Administration console**.
2. Servidor de MariaDB: `localhost`; puerto: `3306`.
3. Apache: conservar sus rutas detectadas, normalmente `/etc/apache2/apache2.conf` y usuario/grupo `www-data`.
4. Usuario de conexión a MariaDB: `ocsuid` y la **contraseña real elegida en el paso 4**, cuando se soliciten. Si el instalador conserva valores predeterminados, corregirlos en el paso 7.
5. Aceptar las rutas predeterminadas de instalación, salvo que el equipo tenga una configuración específica.
6. La API REST es opcional para el objetivo de inventariar una máquina.

El asistente genera un registro llamado `ocs_server_setup.log` en la carpeta desde la que se ejecuta, útil para diagnosticar fallos.

## 7. Configurar y activar Apache

Comprobar los archivos de configuración:

```bash
ls -l /etc/apache2/conf-available/*ocs*
```

Activar la consola web:

```bash
sudo a2enconf ocsinventory-reports
```

Antes de activar el módulo de comunicaciones, verificar que su usuario sea `ocsuid`:

```bash
sudo grep -E '^\s*PerlSetEnv\s+OCS_DB_(HOST|PORT|NAME|USER)' \
  /etc/apache2/conf-available/z-ocsinventory-server.conf
```

Editar si fuera necesario:

```bash
sudoedit /etc/apache2/conf-available/z-ocsinventory-server.conf
```

En ese archivo las líneas importantes deben contener los **valores reales del servidor**:

```apache
PerlSetEnv OCS_DB_HOST localhost
PerlSetEnv OCS_DB_PORT 3306
PerlSetEnv OCS_DB_NAME ocsweb
PerlSetEnv OCS_DB_USER ocsuid
PerlSetVar OCS_DB_PWD CONTRASENA_REAL_NO_PUBLICAR
```

**Importante:** el bloque anterior es orientativo; no pegar una contraseña ficticia en el servidor. La línea de contraseña contiene un secreto que **no debe subirse a GitHub**.

Activar el servicio de comunicación, comprobar la sintaxis y reiniciar:

```bash
sudo a2enconf z-ocsinventory-server
sudo apache2ctl configtest
sudo systemctl restart apache2
sudo systemctl status apache2 --no-pager
```

**Explicación:** `ocsinventory-reports` publica el panel `/ocsreports`; `z-ocsinventory-server` incorpora el endpoint de los agentes `/ocsinventory`. `configtest` detecta errores de sintaxis antes de continuar. Si Apache falla, revisar el apartado de incidencias y desactivar temporalmente solo esta configuración para recuperar Moodle.

## 8. Resolver el error de permisos del instalador web (incidencia real)

En nuestro caso, el instalador mostraba:

```text
ERROR: can't write in directory (on dbconfig.inc.php), please set the required rights in order to install ocsinventory.
```

Primero concedimos **permisos temporales** al usuario de Apache (`www-data`):

```bash
sudo touch /usr/share/ocsinventory-reports/ocsreports/dbconfig.inc.php
sudo chown www-data:www-data /usr/share/ocsinventory-reports/ocsreports
sudo chmod 775 /usr/share/ocsinventory-reports/ocsreports
sudo chown www-data:www-data /usr/share/ocsinventory-reports/ocsreports/dbconfig.inc.php
sudo chmod 660 /usr/share/ocsinventory-reports/ocsreports/dbconfig.inc.php

sudo mkdir -p /var/lib/ocsinventory-reports
sudo chown -R www-data:www-data /var/lib/ocsinventory-reports
sudo chmod 775 /var/lib/ocsinventory-reports
```

La prueba de escritura desde la terminal daba `bool(true)`:

```bash
sudo -u www-data php -r 'var_dump(is_writable("/usr/share/ocsinventory-reports/ocsreports"));'
```

Sin embargo, desde un archivo PHP ejecutado a través de Apache la escritura continuaba bloqueada. Se revisaron las restricciones del servicio:

```bash
systemctl show apache2 -p ProtectSystem -p ReadWritePaths
```

Para permitir temporalmente la escritura en **esa carpeta**, añadimos una excepción de `systemd`:

```bash
sudo mkdir -p /etc/systemd/system/apache2.service.d
printf '[Service]\nReadWritePaths=/usr/share/ocsinventory-reports/ocsreports\n' | \
  sudo tee /etc/systemd/system/apache2.service.d/ocs-install.conf
sudo systemctl daemon-reload
sudo systemctl restart apache2
```

**Resultado observado:** se pudo continuar al formulario de MariaDB y completar la instalación web. Esta excepción está pensada para la **fase de instalación**, no para dejar abierta permanentemente la carpeta.

## 9. Finalizar la instalación desde el navegador

Abrir:

```text
http://192.168.0.103/ocsreports
```

En el formulario de MariaDB introducir:

| Campo | Valor |
|---|---|
| MySQL login | `ocsuid` |
| MySQL password | Contraseña real configurada en MariaDB |
| Name of Database | `ocsweb` |
| MySQL HostName | `localhost` |
| MySQL Port | `3306` |
| SSL | Desactivado para esta prueba local; configurar cifrado para otros entornos |

Enviar el formulario y, si el instalador lo solicita, ejecutar la **actualización de la base de datos**. Durante esta práctica apareció una actualización de la versión interna `7068` a `7077`, tras la que se llegó a la pantalla de inicio de sesión.

Las credenciales iniciales del panel pueden ser `admin` / `admin`. **Cambiar inmediatamente la contraseña si están activas**.

## 10. Verificar funcionamiento

```bash
sudo systemctl is-active apache2
sudo systemctl is-active mariadb
sudo apache2ctl -M 2>/dev/null | grep -E 'php|perl'
sudo mariadb -N -e 'SELECT COUNT(*) FROM ocsweb.hardware;'
```

**Interpretación:** comprobar que ambos servicios están activos y que Apache tiene los módulos cargados. La última consulta cuenta los equipos registrados.

En nuestra práctica, el **panel sí abrió correctamente**, pero el contador seguía en **0 máquinas**: no debe considerarse validado el inventario automático.

## 11. Prueba del agente (pendiente de solucionar)

```bash
sudo apt install -y ocsinventory-agent
sudo ocsinventory-agent --server=http://127.0.0.1/ocsinventory --force
```

**Explicación:** instalamos un agente en la propia máquina virtual y forzamos un envío al servicio `/ocsinventory`. Como el agente y el servidor están en la misma VM, usamos `127.0.0.1`.

**Resultado real:** tras activar el módulo Perl, el agente llegó a enviar una petición de inventario, pero el servidor devolvió **`500 Internal Server Error`** y la tabla `ocsweb.hardware` continuó vacía. El registro de Apache mostró `Can't call method "rollback" on an undefined value`, compatible con un fallo interno que todavía requiere diagnóstico. No afirmar en GitHub que la sincronización del agente se ha completado.

## 12. Incidencias y comandos de diagnóstico

### 12.1. Error `404 Not Found` en `/ocsinventory`

El archivo `z-ocsinventory-server.conf` estaba presente pero no habilitado. Se activó con:

```bash
sudo a2enconf z-ocsinventory-server
sudo apache2ctl configtest
sudo systemctl reload apache2
```

### 12.2. Apache dejó de funcionar por falta de `Switch.pm`

El registro mostraba `Can't locate Switch.pm`. Se solucionó instalando el módulo:

```bash
sudo apt install -y libswitch-perl
perl -MSwitch -e 'print "SWITCH OK\n"'
```

Para **recuperar Apache** si la configuración de comunicaciones impide arrancarlo:

```bash
sudo a2disconf z-ocsinventory-server
sudo apache2ctl configtest
sudo systemctl restart apache2
```

Esto deja temporalmente deshabilitado el servicio de agentes, pero permite recuperar el panel web y Moodle.

### 12.3. Aviso sobre `XML::Entities`

Comprobar si está disponible:

```bash
perl -MXML::Entities -e 'print "XML::Entities OK\n"'
```

Si falta y la funcionalidad SOAP la necesita, la documentación oficial contempla instalarlo mediante CPAN. Es una comprobación adicional, **no una solución demostrada para el error HTTP 500 del agente**.

### 12.4. Error HTTP 500 del agente (no resuelto)

Consultar el registro sin mostrar contraseñas:

```bash
sudo tail -n 50 /var/log/apache2/error.log
sudo journalctl -u apache2 -n 50 --no-pager
sudo mariadb -N -e 'SELECT COUNT(*) FROM ocsweb.hardware;'
```

Confirmar también que `OCS_DB_USER` sea `ocsuid` y que la contraseña del servidor de comunicaciones coincida con la de MariaDB.

## 13. Seguridad y limpieza posteriores

**Solo cuando la instalación web esté finalizada**, cambiar la contraseña inicial del panel, eliminar el instalador y retirar los permisos temporales:

```bash
sudo rm -f /usr/share/ocsinventory-reports/ocsreports/install.php
sudo rm -f /usr/share/ocsinventory-reports/ocsreports/ocs_diag.php
sudo chown root:www-data /usr/share/ocsinventory-reports/ocsreports/dbconfig.inc.php
sudo chmod 640 /usr/share/ocsinventory-reports/ocsreports/dbconfig.inc.php
sudo chown root:www-data /usr/share/ocsinventory-reports/ocsreports
sudo chmod 755 /usr/share/ocsinventory-reports/ocsreports
```

Retirar **solo si se creó para esta instalación** la excepción temporal del servicio:

```bash
sudo rm -f /etc/systemd/system/apache2.service.d/ocs-install.conf
sudo systemctl daemon-reload
sudo apache2ctl configtest
sudo systemctl restart apache2
```

Tras retirar la excepción, **verificar de nuevo** el panel y Moodle. Si algún componente falla, investigar qué directorio necesita escritura y conceder el acceso mínimo necesario. Nunca utilizar `chmod -R 777` como solución permanente.

No subir a GitHub archivos que contengan credenciales reales (`dbconfig.inc.php`, `z-ocsinventory-server.conf` sin anonimizar o copias de seguridad de estos archivos).

## 14. Resultado y estado del TFG

| Componente | Estado observado |
|---|---|
| Servidor Apache | ✅ Instalado y operativo |
| Base de datos MariaDB | ✅ Configurada (`ocsweb`) |
| Consola OCS (`/ocsreports`) | ✅ Accesible |
| Error de permisos de instalación | ✅ Solucionado |
| Módulo Perl `Switch` | ✅ Instalado |
| Endpoint y agente (`/ocsinventory`) | ⚠️ Respuesta HTTP 500 en la última prueba |
| Equipos inventariados | ⚠️ 0; falta resolver la sincronización |
| Endurecimiento final de seguridad | ⚠️ Pendiente de comprobar |

## Fuentes oficiales

- [Documentación: instalación del servidor OCS Inventory](https://wiki.ocsinventory-ng.org/03.Basic-documentation/Setting-up-a-OCS-Inventory-Server/)
- [Requisitos y módulos de OCS Inventory](https://wiki.ocsinventory-ng.org/01.Prerequisites/Libraries-version/)
- [Documentación: agentes Unix](https://wiki.ocsinventory-ng.org/03.Basic-documentation/Setting-up-the-UNIX-agent-manually-on-client-computers/)
- [Paquete oficial OCS Inventory NG 2.12.1](https://github.com/OCSInventory-NG/OCSInventory-ocsreports/releases/tag/2.12.1)

---

> **Nota de reproducibilidad:** estos pasos documentan la instalación de laboratorio. No se deben volver a ejecutar indiscriminadamente sobre una instalación ya existente: `CREATE DATABASE`, `CREATE USER`, `setup.sh` y las modificaciones de permisos pueden alterar una configuración funcional.
