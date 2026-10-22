# Instalación y configuración de Zabbix 7.4 en Ubuntu 26.04 — TFG

## 1. Objetivo

Instalar y configurar **Zabbix Server 7.4**, su interfaz web y **Zabbix Agent 2** en una máquina virtual Ubuntu Server 26.04.1 LTS, para supervisar CPU, RAM, disco y disponibilidad del servidor. Se reutilizan Apache y MariaDB, ya presentes en el proyecto por Moodle y OCS Inventory NG.

**Resultado obtenido:** Zabbix Server y Agent 2 activos, frontend accesible e inicio de sesión realizado; host `Zabbix server` disponible y métricas recientes visibles. Se comprobó la gráfica **Available memory in %**. Quedaron 11 elementos marcados *Not supported*, pendientes de revisar si se desea una instalación totalmente depurada.

> **Seguridad:** los valores `<CONTRASENA_BD_SEGURA>` y `<NUEVA_CONTRASENA_ADMIN>` son ejemplos, no contraseñas reales. No subir secretos a GitHub. Los comandos SQL de creación/importación son para una instalación **nueva**: no ejecutarlos sobre una base de datos con datos útiles.

## 2. Entorno utilizado

| Elemento | Valor |
|---|---|
| Hipervisor | VirtualBox |
| Sistema operativo | Ubuntu Server 26.04.1 LTS (`resolute`), x86_64 |
| Versión instalada | Zabbix 7.4.16 |
| Base de datos | MariaDB 11.8.6 |
| Web | Apache 2.4.66 y PHP 8.5 |
| Servicios | `zabbix-server`, `zabbix-agent2`, `apache2`, `mariadb` |
| URL usada desde Windows | `http://127.0.0.1:8080/zabbix/` |

En VirtualBox se mantuvo el adaptador puente y se añadió un segundo adaptador NAT con DHCP para trabajar también desde una red que no asignaba IPv4 a la interfaz en puente. Se configuró el reenvío TCP `127.0.0.1:2222 → invitado:22` para SSH y `127.0.0.1:8080 → invitado:80` para Apache. Para conectarse desde Windows:

```powershell
ssh -p 2222 vboxuser@127.0.0.1
```

## 3. Verificaciones previas

```bash
lsb_release -a
uname -m
systemctl is-active apache2
systemctl is-active mariadb
php -v | head -n 1
dpkg -l | grep zabbix
```

Apache y MariaDB debían estar activos, y se confirmó que Zabbix todavía no estaba instalado.

## 4. Configurar el repositorio oficial de Zabbix

```bash
cd /tmp
wget https://repo.zabbix.com/zabbix/7.4/release/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest_7.4+ubuntu26.04_all.deb
sudo dpkg -i zabbix-release_latest_7.4+ubuntu26.04_all.deb
sudo apt update
apt-cache policy zabbix-server-mysql
```

La consulta de APT mostró una versión candidata `1:7.4.16-1+ubuntu26.04` en el repositorio oficial.

### Incidencia A: dependencias PHP incompatibles

La primera instalación falló porque el repositorio PPA `ondrej/php` para Ubuntu **24.04 (`noble`)** estaba activo en una máquina Ubuntu **26.04 (`resolute`)**. APT intentaba elegir paquetes `8.5.12-...ubuntu24.04...deb.sury.org` en lugar de la versión nativa de Ubuntu 26.04. El repositorio no estaba solo en `sources.list.d`, sino también en **`/etc/apt/sources.list`**.

Para diagnosticarlo:

```bash
apt-cache policy php8.5-fpm php8.5-common
sudo grep -RniE 'ondrej|sury|noble' /etc/apt/sources.list /etc/apt/sources.list.d/ 2>/dev/null
```

Se hizo copia del archivo y se comentó **únicamente la línea de ese PPA**:

```bash
sudo cp /etc/apt/sources.list /etc/apt/sources.list.bak
sudo sed -i '/^[[:space:]]*deb .*ppa.launchpadcontent.net\/ondrej\/php/ s/^/# /' /etc/apt/sources.list
sudo apt update
apt-cache policy php8.5-fpm php8.5-common
```

Se verificó que los candidatos ya procedían de Ubuntu 26.04 (`8.5.4-0ubuntu1.5`). Se simuló la instalación antes de aplicarla:

```bash
sudo apt install -s php8.5-fpm php8.5-bcmath php8.5-ldap fonts-dejavu-core \
  zabbix-server-mysql zabbix-frontend-php zabbix-apache-conf \
  zabbix-sql-scripts zabbix-agent2
```

El resumen de la simulación fue **22 paquetes nuevos, 12 actualizaciones, 0 eliminaciones**. Se revisó que no se desinstalasen servicios de Moodle u OCS.

## 5. Instalar los paquetes

```bash
sudo apt install -y \
  php8.5-fpm php8.5-bcmath php8.5-ldap fonts-dejavu-core \
  zabbix-server-mysql zabbix-frontend-php zabbix-apache-conf \
  zabbix-sql-scripts zabbix-agent2

dpkg -l | grep '^ii.*zabbix'
```

- `zabbix-server-mysql`: servicio central que procesa la monitorización.
- `zabbix-frontend-php`: interfaz web.
- `zabbix-apache-conf`: configuración para Apache.
- `zabbix-sql-scripts`: esquema y datos iniciales.
- `zabbix-agent2`: recoge métricas del servidor Ubuntu.

### Incidencia B: Apache no arrancaba después de instalar Zabbix

El registro mostró:

```text
Syntax error on line 7 of /etc/apache2/conf-enabled/zabbix.conf:
Invalid command 'ProxyTimeout'
```

Se habilitaron los módulos necesarios y se verificó la sintaxis:

```bash
sudo a2enmod proxy
sudo a2enmod proxy_fcgi
sudo apache2ctl configtest
sudo systemctl restart apache2
systemctl is-active apache2
systemctl is-active mariadb
```

El estado de Apache y MariaDB volvió a ser `active`. **No se activó globalmente PHP-FPM**, para no alterar de forma innecesaria las otras aplicaciones de Apache.

## 6. Crear la base de datos de Zabbix

Entrar en MariaDB:

```bash
sudo mariadb
```

Crear una base de datos independiente y un usuario dedicado:

```sql
CREATE DATABASE zabbix CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;
CREATE USER 'zabbix'@'localhost' IDENTIFIED BY '<CONTRASENA_BD_SEGURA>';
GRANT ALL PRIVILEGES ON zabbix.* TO 'zabbix'@'localhost';
EXIT;
```

Verificación:

```bash
sudo mariadb -e "SHOW DATABASES LIKE 'zabbix';"
```

### Importar esquema y datos iniciales

Para una base de datos recién creada y **vacía**, habilitar temporalmente la creación de funciones y triggers si es necesario:

```bash
sudo mariadb -e "SET GLOBAL log_bin_trust_function_creators = 1;"
```

Importación (pedirá la contraseña del usuario de base de datos):

```bash
set -o pipefail
zcat /usr/share/zabbix/sql-scripts/mysql/server.sql.gz | \
  mariadb --default-character-set=utf8mb4 -u zabbix -p zabbix
```

Comprobar **inmediatamente** el código de salida, antes de ejecutar otro comando:

```bash
echo $?
```

`0` indica que la importación terminó correctamente. Se confirmó también con:

```bash
sudo mariadb -e "SELECT COUNT(*) AS tablas FROM information_schema.tables WHERE table_schema='zabbix';"
sudo mariadb -e "SELECT * FROM zabbix.dbversion;"
sudo mariadb -e "SET GLOBAL log_bin_trust_function_creators = 0;"
```

En la instalación se crearon **207 tablas**, y la importación final devolvió `0`.

### Incidencia C: importación interrumpida y tabla duplicada

La importación se interrumpió y, al repetirla, apareció `ERROR 1050: Table 'role' already exists`. La causa fue que el esquema ya estaba parcialmente creado. También apareció una vez `ERROR 1045: Access denied` por credenciales de MariaDB.

Para una instalación **sin datos de monitorización previos**, se detuvo Zabbix, se recreó **solo** su base de datos y se comprobó que tenía **0 tablas** antes de importarla de nuevo. No repetir estos comandos sobre una instalación ya en uso:

```bash
sudo systemctl stop zabbix-server
sudo mariadb
```

```sql
DROP DATABASE IF EXISTS zabbix;
CREATE DATABASE zabbix CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;
GRANT ALL PRIVILEGES ON zabbix.* TO 'zabbix'@'localhost';
EXIT;
```

En el laboratorio, cuando la contraseña no coincidía, se actualizó con `ALTER USER` y se verificó el acceso con `mariadb -u zabbix -p zabbix` antes de repetir la importación. **No publicar la contraseña utilizada.**

## 7. Configurar Zabbix Server

Se conservó el archivo de configuración original `/etc/zabbix/zabbix_server.conf`, haciendo copia de seguridad y configurando los parámetros:

```ini
DBHost=localhost
DBName=zabbix
DBUser=zabbix
DBPassword=<CONTRASENA_BD_SEGURA>
```

El valor de `DBPassword` debe coincidir con la contraseña del usuario MariaDB `zabbix`. Para proteger el archivo:

```bash
sudo chown root:zabbix /etc/zabbix/zabbix_server.conf
sudo chmod 640 /etc/zabbix/zabbix_server.conf
sudo systemctl enable --now zabbix-server
sudo systemctl restart zabbix-server
sudo systemctl status zabbix-server --no-pager
```

Se comprobó que `zabbix-server` aparecía como `active (running)`.

## 8. Activar Zabbix Agent 2 y verificar servicios

```bash
sudo systemctl enable --now zabbix-agent2
systemctl is-active zabbix-server
systemctl is-active zabbix-agent2
systemctl is-active apache2
systemctl is-active mariadb
sudo ss -ltnp | grep -E ':10050|:10051|:80 '
curl -I http://127.0.0.1/zabbix/
```

Los cuatro servicios figuraron como **activos**. Los puertos `10050` (agente), `10051` (servidor) y `80` (Apache) estaban escuchando. La URL local devolvió `HTTP/1.1 302 Found` hacia `setup.php`, señal de que se podía iniciar el asistente web.

## 9. Asistente web

En Windows se accedió a:

```text
http://127.0.0.1:8080/zabbix/
```

En el asistente de Zabbix 7.4:

1. Se dejó **English**, al no aparecer español en la selección inicial.
2. Se superó **Check of pre-requisites**.
3. En **Configure DB connection**: `MySQL`, host `localhost`, puerto `0` (predeterminado), base de datos `zabbix`, usuario `zabbix` y la contraseña privada correspondiente.
4. En **Settings**, se estableció el nombre `Zabbix - TFG Paiporta` y se utilizó `Europe/Madrid` como zona horaria; se mantuvo el tema `Blue` y no se habilitó el cifrado web en este laboratorio local.
5. Se revisó **Pre-installation summary** y apareció **“You have successfully installed Zabbix frontend”**.

> Para un entorno real debe configurarse HTTPS. En este laboratorio se accedió por HTTP a través de una redirección NAT local.

### Incidencia D: inicio de sesión

El usuario de MariaDB `zabbix` **no** es el administrador web. El inicio de sesión correcto utilizó el usuario **`Admin` (A mayúscula)**. Tras una confusión con el nombre y el bloqueo por intentos fallidos se logró entrar en el panel. Las credenciales iniciales deben sustituirse por otras privadas antes de exponer el servicio.

## 10. Verificación final de monitorización

Desde **Monitoring → Latest data**, se comprobó el host **`Zabbix server`**, con indicadores de CPU y RAM actualizados y enlaces a sus gráficas.

Evidencias obtenidas:

- Dashboard general: el servidor aparece disponible, con indicadores de CPU y memoria.
- Página **Latest data**: se mostraron **128 elementos**, de los cuales **117** estaban normales y **11** indicaban *Not supported*.
- Gráfica **Available memory in %**: valores mostrados alrededor de **67,53 % a 68,59 %** durante la ventana visualizada. Confirma recolección y representación histórica de métricas.

### Capturas recomendadas para documentar

- Repositorio instalado y paquetes de Zabbix.
- Servicios `active (running)`.
- Asistente web completado.
- Dashboard con disponibilidad del host.
- **Monitoring → Latest data**.
- Gráfica de memoria RAM; opcionalmente CPU y disco.

## 11. Pendientes y seguridad

La **instalación y monitorización básica están terminadas**, pero conviene:

- Cambiar la contraseña predeterminada del usuario web `Admin`.
- Cambiar cualquier contraseña de ejemplo de MariaDB y actualizar `DBPassword` en `zabbix_server.conf`.
- Mantener los permisos restringidos sobre archivos con credenciales; **no subirlos a GitHub**.
- Revisar los 11 ítems *Not supported* para depurar la plantilla de monitorización.
- Configurar HTTPS si se accede desde una red distinta del propio equipo.
- Revisar que Moodle y OCS sigan respondiendo tras la actualización de PHP.
- Continuar más adelante con el problema pendiente del agente de OCS Inventory.

## 12. Resultado

Se desplegaron correctamente Zabbix 7.4.16, su servidor, frontend y agente 2 en Ubuntu 26.04.1 LTS con Apache y MariaDB. Se solucionaron conflictos de repositorios PHP y de módulos de Apache; se creó e importó la base de datos, se completó el asistente web y se verificó la visualización de métricas en tiempo real e históricas.
