# OCS Inventory NG — Solución al error de permisos durante la instalación

## Contexto

Durante la instalación de **OCS Inventory NG 2.12.1** en una máquina virtual **Ubuntu**, con **Apache** ya instalado para alojar Moodle, el asistente web de OCS no permitía avanzar porque no podía escribir su archivo de configuración de la base de datos.

El panel de instalación se accedía desde `http://IP_DEL_SERVIDOR/ocsreports`.

## 1. Problema detectado

El asistente mostraba un error similar a:

```text
ERROR: can't write in directory (on dbconfig.inc.php), please set the required rights in order to install ocsinventory.
```

También avisaba de que `/var/lib/ocsinventory-reports` debía tener permisos de escritura.

## 2. Primera comprobación: permisos de Linux

Se prepararon las rutas que OCS necesita durante la instalación y se concedieron permisos temporales al usuario de Apache (`www-data`):

```bash
sudo touch /usr/share/ocsinventory-reports/ocsreports/dbconfig.inc.php
sudo chown www-data:www-data /usr/share/ocsinventory-reports/ocsreports/dbconfig.inc.php
sudo chmod 660 /usr/share/ocsinventory-reports/ocsreports/dbconfig.inc.php

sudo chown www-data:www-data /usr/share/ocsinventory-reports/ocsreports
sudo chmod 775 /usr/share/ocsinventory-reports/ocsreports

sudo mkdir -p /var/lib/ocsinventory-reports
sudo chown -R www-data:www-data /var/lib/ocsinventory-reports
sudo chmod 775 /var/lib/ocsinventory-reports

sudo systemctl restart apache2
```

Después se comprobó la escritura desde PHP ejecutado en la terminal:

```bash
sudo -u www-data php -r 'var_dump(is_writable("/usr/share/ocsinventory-reports/ocsreports"));'
sudo -u www-data php -r 'var_dump(is_writable("/usr/share/ocsinventory-reports/ocsreports/dbconfig.inc.php"));'
```

**Ambas comprobaciones devolvieron `bool(true)`, pero el error seguía apareciendo en el navegador.** Por tanto, los permisos tradicionales del sistema de archivos no explicaban por sí solos el problema.

## 3. Diagnóstico: diferencia entre PHP de terminal y PHP de Apache

Se creó temporalmente `ocs_diag.php` dentro de `ocsreports` con una prueba de `is_writable()` para las dos rutas. Al visitarlo desde el navegador, PHP ejecutado mediante Apache devolvía `ERROR` en ambas pruebas, mientras que PHP ejecutado desde la terminal devolvía `true`.

Esto indicó que **Apache tenía una restricción de escritura adicional**. Se investigó la configuración de aislamiento del servicio mediante `systemd` (`ProtectSystem` / `ReadWritePaths`). No se conservó en el diagnóstico el valor exacto de `ProtectSystem`, por lo que no se atribuye el fallo a un valor concreto sin verificarlo.

Comando de diagnóstico útil:

```bash
systemctl show apache2 -p ProtectSystem -p ReadWritePaths
```

## 4. Solución aplicada: excepción de escritura para OCS

Se creó un ajuste específico del servicio Apache para permitir la escritura en la carpeta de instalación de OCS:

```bash
sudo mkdir -p /etc/systemd/system/apache2.service.d

printf '[Service]\nReadWritePaths=/usr/share/ocsinventory-reports/ocsreports\n' | \
  sudo tee /etc/systemd/system/apache2.service.d/ocs-install.conf

sudo systemctl daemon-reload
sudo systemctl restart apache2
```

A diferencia de desactivar globalmente las protecciones de Apache, esta excepción se limita a una ruta concreta.

Después se volvió a comprobar la página de OCS. **El asistente permitió continuar con la configuración de MariaDB**, se completó el paso de actualización de la base de datos y se consiguió acceder al panel web de administración.

## 5. Resultado

- Panel web de **OCS Inventory NG accesible**.
- **Conexión y actualización de la base de datos** realizadas durante el asistente.
- **Error inicial de escritura en `dbconfig.inc.php` solucionado**.
- **Inventario automático de agentes: pendiente**. Al finalizar estas pruebas, el panel seguía mostrando **0 máquinas**. Este es un problema posterior, distinto del fallo de permisos de instalación.

## 6. Seguridad y tareas pendientes

Los permisos ampliados y la excepción `ReadWritePaths` se aplicaron para **completar la instalación**. No deben considerarse la configuración definitiva sin una revisión de seguridad. Una vez verificado el funcionamiento, se recomienda:

1. Eliminar el fichero temporal de diagnóstico `ocs_diag.php`, si aún existe.
2. Eliminar o deshabilitar `install.php` **solo después** de finalizar la instalación y comprobar que no se necesita.
3. Cambiar la contraseña inicial del administrador web, si sigue siendo la predeterminada.
4. Devolver `dbconfig.inc.php` a permisos de solo lectura para el proceso web cuando la instalación ya no necesite modificarlo.
5. Revisar si puede retirarse la excepción temporal `ocs-install.conf` sin afectar al funcionamiento; comprobar el estado de Apache y OCS después.
6. No subir contraseñas de MariaDB ni archivos de configuración con credenciales al repositorio.

> **Nota:** Las medidas de esta sección son recomendaciones **pendientes de comprobación**, no acciones que se den por realizadas. Se deben aplicar de forma controlada porque Apache también aloja Moodle.

## 7. Comandos de verificación

```bash
sudo systemctl status apache2 --no-pager
sudo apache2ctl configtest
systemctl show apache2 -p ProtectSystem -p ReadWritePaths
```

**Conclusión:** el problema no era únicamente de permisos UNIX (`chown`/`chmod`); el comportamiento diferente entre PHP de terminal y PHP servido por Apache permitió localizar una restricción adicional. La excepción de escritura configurada en `systemd` permitió terminar la instalación web de OCS Inventory.
