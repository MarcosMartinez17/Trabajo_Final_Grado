# Resumen del Proceso de Instalación y Despliegue de Moodle (v4.3.2) en Ubuntu

Este documento recopila los pasos realizados, los problemas encontrados y los comandos utilizados para configurar el entorno de Moodle en la máquina virtual Ubuntu orientada al desarrollo del TFG.

---

## 1. Contexto del Entorno
* **Servidor Web:** Apache2
* **Ruta web raíz:** `/var/www/html/`
* **Directorio final de Moodle:** `/var/www/html/moodle/`
* **Dirección IP del servidor:** `192.168.0.104`

---

## 2. Pasos y Comandos Ejecutados

### Opción A: Transferencia manual desde Windows (vía SCP)
Inicialmente se intentó transferir el archivo comprimido desde la máquina anfitriona hacia la máquina virtual utilizando el protocolo `scp`.

1. **Subida del archivo a la máquina virtual (PowerShell de Windows):**
   ```powershell
   scp "C:\Users\Crism\Downloads\moodle-4.3.2.zip" vboxuser@192.168.0.104:/home/vboxuser/
   ```

2. **Movimiento y preparación del paquete (Terminal SSH - Ubuntu):**
   ```bash
   sudo mv /home/vboxuser/moodle-4.3.2.zip /var/www/html/
   cd /var/www/html
   sudo unzip -q moodle-4.3.2.zip
   ```

### Opción B: Descarga directa en el servidor (Método más eficiente)
Para evitar conflictos de rutas y transferencias repetitivas, se optó por descargar el código fuente directamente desde el repositorio oficial de GitHub hacia el servidor web mediante `wget`.

```bash
cd /var/www/html 
sudo rm -rf moodle moodle-4.3.2.zip 
sudo wget https://github.com/moodle/moodle/archive/refs/tags/v4.3.2.zip -O moodle-4.3.2.zip 
sudo unzip -q moodle-4.3.2.zip 
sudo mv moodle-4.3.2 moodle 
sudo rm -f moodle-4.3.2.zip
```

---

## 3. Configuración de Permisos y Servicios
Una vez descomprimido y renombrado el directorio como `moodle`, fue necesario ajustar los permisos para que el servidor web Apache (`www-data`) pudiera interpretar los archivos correctamente y reiniciar el servicio.

```bash
sudo chown -R www-data:www-data moodle 
sudo chmod -R 755 moodle 
sudo systemctl restart apache2
```

---

## 4. Verificación y Acceso
Finalmente, se comprobó el correcto funcionamiento accediendo desde el navegador web del entorno local a la dirección configurada:

* **URL de acceso:** `http://192.168.0.104/moodle/`
* **Resultado:** Visualización exitosa de la pantalla de bienvenida e inicio del asistente de instalación de Moodle.
