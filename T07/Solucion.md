**T02: Missió Apache** (con extra)

> **“Esta práctica no consiste en crear una página web, sino en configurar un servidor web profesional completo con Apache, multisitio, HTTPS, certificados SSL, error personalizado y soporte HTTP/2.”**

**Autor:** Santiago · **Lista 10**\
**Entorno:** Ubuntu Server (VM) + VirtualBox\
**Estado:** Completada con éxito y verificada con pruebas y capturas\
**Extra incluido:** Acceso desde cliente Windows en red **Host‑Only**

***

## 🗺️ Índice

1.  \#-objetivo-de-la-práctica
2.  \#-arquitectura-y-máquinas
3.  \#-redes-y-adaptadores-virtualbox
4.  \#-fase-1--instalación-y-configuración-base
5.  \#-fase-2--virtualhosts-multidominio
6.  \#-fase-3--error-404-personalizado
7.  \#-fase-4--sslhttps-con-san
8.  \#-fase-5--optimización-http2
9.  \#-pruebas-con-curl-y-navegador
10. \#-checklist-de-capturas-obligatorias
11. \#-solución-de-problemas-comunes
12. \#-extra--cliente-windows-en-red-hostonly
13. \#-apéndice--archivos-de-configuración-finales

***

## Objetivo de la práctica

Configurar en **una sola máquina Ubuntu Server** un **servidor Apache 2.4** con:

*   **Multidominio** (VirtualHosts): `projectenexus10.test` y `academia10.test`
*   **Error 404 personalizado**
*   **SSL/HTTPS** con certificado **autosignado** (RSA 2048 · 365 días · **SAN** para ambos dominios)
*   **Redirección** de **HTTP → HTTPS**
*   **HTTP/2** activado y comprobado

***

## Arquitectura y máquinas

**Servidor (VM 1):** Ubuntu Server con Apache

*   Dominios locales:
    *   `projectenexus10.test`
    *   `academia10.test`
*   Rutas web:
    *   `/var/www/projectenexus10.test/public_html`
    *   `/var/www/academia10.test/public_html`
*   IP Host‑Only (ejemplo real de esta práctica): `192.168.56.117`

**Cliente (VM 2) — *(solo para el EXTRA)*:** Windows 11 “windows FTP”

*   IP Host‑Only (ejemplo real): `192.168.56.125`
*   Acceso a los dominios del servidor vía **/etc/hosts de Windows**.

> **Nota:** La práctica oficial **no exige** el cliente. El **EXTRA** es para demostrar acceso desde otra máquina.

***

## Redes y adaptadores VirtualBox

Para el **servidor Ubuntu**:

*   **Adaptador 1:** NAT (salida a Internet)
*   **Adaptador 2:** Host‑Only (laboratorio local) → IP 192.168.56.X

Para el **cliente Windows 11** *(EXTRA)*:

*   **Adaptador 1:** NAT
*   **Adaptador 2:** Host‑Only → IP 192.168.56.Y

> **Comprobación en Ubuntu:**

```
ip a
```

> **Comprobación en Windows (PowerShell):**

```
ipconfig
```

***

## Fase 1 — Instalación y configuración base

### 1) Instalar Apache2

```
sudo apt update && sudo apt install -y apache2
```

### 2) Verificar servicio y configuración

```
systemctl status apache2
```
```
sudo apachectl configtest
```

> Si aparece el aviso del FQDN, definir `ServerName` global:

```
echo 'ServerName localhost' | sudo tee /etc/apache2/conf-available/servername.conf
```

```
sudo a2enconf servername && sudo apachectl configtest && sudo systemctl reload apache2
```

### 3) Confirmar usuario y permisos web

```
ls -ld /var/www
```
```
ps aux | grep apache2
```

### 4) UFW — permitir HTTP/HTTPS

```
sudo ufw allow "Apache Full" && sudo ufw enable && sudo ufw status
```

***

## Fase 2 — VirtualHosts multidominio

### 1) Estructura de directorios

```
sudo mkdir -p /var/www/projectenexus10.test/public_html
```
```
sudo mkdir -p /var/www/academia10.test/public_html
```
```
sudo chown -R www-data:www-data /var/www/projectenexus10.test /var/www/academia10.test
```
```
sudo chmod -R 755 /var/www
```

### 2) Páginas de prueba

```
echo "<h1>Projecte Nexus 10 OK</h1>" | sudo tee /var/www/projectenexus10.test/public_html/index.html
```
```
echo "<h1>Academia 10 OK</h1>" | sudo tee /var/www/academia10.test/public_html/index.html
```

### 3) VirtualHost HTTP — `projectenexus10.test`

```
sudo nano /etc/apache2/sites-available/projectenexus10.test.conf 
```

<VirtualHost *:80>
    ServerName projectenexus10.test
    DocumentRoot /var/www/projectenexus10.test/public_html
    <Directory /var/www/projectenexus10.test/public_html>
        AllowOverride All
        Require all granted
    </Directory>
    ErrorLog ${APACHE_LOG_DIR}/projectenexus10_error.log
    CustomLog ${APACHE_LOG_DIR}/projectenexus10_access.log combined
</VirtualHost>
EOF
```

### 4) VirtualHost HTTP — `academia10.test`

```
sudo nano /etc/apache2/sites-available/academia10.test.conf
```

<VirtualHost *:80>
    ServerName academia10.test
    DocumentRoot /var/www/academia10.test/public_html
    <Directory /var/www/academia10.test/public_html>
        AllowOverride All
        Require all granted
    </Directory>
    ErrorLog ${APACHE_LOG_DIR}/academia10_error.log
    CustomLog ${APACHE_LOG_DIR}/academia10_access.log combined
</VirtualHost>
EOF
```

### 5) Activar sitios y recargar

```
sudo a2ensite projectenexus10.test.conf academia10.test.conf && sudo apachectl configtest && sudo systemctl reload apache2
```

### 6) Resolver dominios en **Ubuntu Server** (`/etc/hosts`)

```
sudo bash -lc 'printf "127.0.0.1\tprojectenexus10.test\n127.0.0.1\tacademia10.test\n" >> /etc/hosts'
```

### 7) Probar con `curl`

```
curl http://projectenexus10.test
```
```
curl http://academia10.test
```

***

## Fase 3 — Error 404 personalizado

### 1) Crear el archivo (lo editas a mano)

```
sudo mkdir -p /var/www/projectenexus10.test/public_html/errors
```
```
sudo touch /var/www/projectenexus10.test/public_html/errors/404.html
```
```
sudo nano /var/www/projectenexus10.test/public_html/errors/404.html
```

> **Plantilla sugerida (para pegar dentro de nano):**



### 2) Asociar el ErrorDocument **en HTTP** (80)

```
sudo sed -i '/DocumentRoot/a \ \ \ \ ErrorDocument 404 /errors/404.html' /etc/apache2/sites-available/projectenexus10.test.conf
```
```
sudo apachectl configtest && sudo systemctl reload apache2
```

### 3) Probar el 404 (antes de SSL)

```
curl -i http://projectenexus10.test/no-existe
```

***

## Fase 4 — SSL/HTTPS con SAN

### 1) Activar módulo SSL

```
sudo a2enmod ssl 
```
```
sudo systemctl reload apache2
```

### 2) Config de OpenSSL con **SAN** (ambos dominios)

```
sudo nano /tmp/openssl-san.conf

[ req ]
default_bits       = 2048
prompt             = no
default_md         = sha256
req_extensions     = req_ext
distinguished_name = dn
[ dn ]
CN = projectenexus10.test
[ req_ext ]
subjectAltName = @alt_names
[ alt_names ]
DNS.1 = projectenexus10.test
DNS.2 = academia10.test
EOF
```

### 3) Generar certificado autosignado (RSA 2048 · 365 días)

```
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/private/apache-selfsigned.key -out /etc/ssl/certs/apache-selfsigned.crt -config /tmp/openssl-san.conf
```

### 4) VirtualHost HTTPS — `projectenexus10.test`

```bash
sudo tee /etc/apache2/sites-available/projectenexus10-ssl.conf >/dev/null <<'EOF'
<IfModule mod_ssl.c>
<VirtualHost *:443>
    ServerName projectenexus10.test
    DocumentRoot /var/www/projectenexus10.test/public_html
    <Directory /var/www/projectenexus10.test/public_html>
        AllowOverride All
        Require all granted
    </Directory>
    # IMPORTANTE: 404 también en HTTPS
    ErrorDocument 404 /errors/404.html
    SSLEngine on
    SSLCertificateFile /etc/ssl/certs/apache-selfsigned.crt
    SSLCertificateKeyFile /etc/ssl/private/apache-selfsigned.key
    ErrorLog ${APACHE_LOG_DIR}/projectenexus10_ssl_error.log
    CustomLog ${APACHE_LOG_DIR}/projectenexus10_ssl_access.log combined
</VirtualHost>
</IfModule>
EOF
```

### 5) VirtualHost HTTPS — `academia10.test`

```bash
sudo tee /etc/apache2/sites-available/academia10-ssl.conf >/dev/null <<'EOF'
<IfModule mod_ssl.c>
<VirtualHost *:443>
    ServerName academia10.test
    DocumentRoot /var/www/academia10.test/public_html
    <Directory /var/www/academia10.test/public_html>
        AllowOverride All
        Require all granted
    </Directory>
    SSLEngine on
    SSLCertificateFile /etc/ssl/certs/apache-selfsigned.crt
    SSLCertificateKeyFile /etc/ssl/private/apache-selfsigned.key
    ErrorLog ${APACHE_LOG_DIR}/academia10_ssl_error.log
    CustomLog ${APACHE_LOG_DIR}/academia10_ssl_access.log combined
</VirtualHost>
</IfModule>
EOF
```

### 6) Activar sitios HTTPS y recargar

```bash
sudo a2ensite projectenexus10-ssl.conf academia10-ssl.conf && sudo apachectl configtest && sudo systemctl reload apache2
```

### 7) Redirección **HTTP → HTTPS** (en ambos sitios HTTP)

```bash
sudo sed -i '/<\/VirtualHost>/i \ \ \ \ Redirect "/" "https://projectenexus10.test/"' /etc/apache2/sites-available/projectenexus10.test.conf
sudo sed -i '/<\/VirtualHost>/i \ \ \ \ Redirect "/" "https://academia10.test/"' /etc/apache2/sites-available/academia10.test.conf
sudo apachectl configtest && sudo systemctl reload apache2
```

***

## Fase 5 — Optimización HTTP/2

### 1) Activar módulo y configurar `Protocols`

```bash
sudo a2enmod http2 && sudo systemctl reload apache2
```

Añadir `Protocols` a ambos **VirtualHost HTTPS**:

*   `projectenexus10-ssl.conf`
*   `academia10-ssl.conf`

```bash
sudo sed -i '/DocumentRoot/a \ \ \ \ Protocols h2 h2c http/1.1' /etc/apache2/sites-available/projectenexus10-ssl.conf
sudo sed -i '/DocumentRoot/a \ \ \ \ Protocols h2 h2c http/1.1' /etc/apache2/sites-available/academia10-ssl.conf
sudo apachectl configtest && sudo systemctl reload apache2
```

***

## 🔎 Pruebas con `curl` y navegador

### HTTPS en ambos dominios (respuesta 200)

```bash
curl -I --http2 -k https://projectenexus10.test
curl -I --http2 -k https://academia10.test
```

### 404 personalizado en HTTPS

```bash
curl -i --http2 -k https://projectenexus10.test/no-existe
```

### Navegador (Firefox/Chrome)

*   Entrar a `https://projectenexus10.test`
*   Abrir **DevTools → Network → columna Protocol** → debe mostrar **h2**
*   Probar `https://projectenexus10.test/no-existe` → **404 personalizado**

***

## Checklist de capturas obligatorias

1.  `apachectl configtest` → **Syntax OK**
2.  `systemctl status apache2` → **active (running)**
3.  `ls -l /var/www` con directorios correctos
4.  `/etc/hosts` (Ubuntu) con ambos dominios
5.  `curl http://...` (fase 2)
6.  Contenido de `404.html`
7.  VirtualHost con `ErrorDocument 404 ...` (HTTP y **HTTPS**)
8.  Generación del certificado (OpenSSL)
9.  `curl -I --http2 -k https://...` (**HTTP/2 200**)
10. (EXTRA) Navegador en Windows mostrando:

*   `https://projectenexus10.test` correcto
*   `https://projectenexus10.test/no-existe` con **404 personalizado**

***

## Solución de problemas comunes

*   **Warning FQDN en `configtest`**\
    Solución: `ServerName localhost` en `conf-available/servername.conf`.

*   **Error de sintaxis en VirtualHost**\
    Mensajes como *“directive missing closing ‘>’”* → revisar comillas y `</VirtualHost>`.

*   **404 personalizado no aparece en HTTPS**\
    **Causa:** `ErrorDocument 404` agregado solo al VirtualHost HTTP (80).\
    **Solución:** añadir también al VirtualHost **HTTPS (443)**.

*   **Desde Windows no resuelve los dominios** *(EXTRA opcional)*\
    Añadir en **hosts de Windows** la IP Host‑Only del servidor:\
    `C:\Windows\System32\drivers\etc\hosts`
    192.168.56.117    projectenexus10.test
    192.168.56.117    academia10.test

***

## ⭐ EXTRA — Cliente Windows en red Host‑Only

**Objetivo:** Simular un entorno real donde un cliente Windows accede al servidor Ubuntu en red privada.

### 1) Ver conectividad

```powershell
ping 192.168.56.117
```

### 2) Hosts de Windows (como admin)

Ruta: `C:\Windows\System32\drivers\etc\hosts`\
Contenido a añadir:

    192.168.56.117    projectenexus10.test
    192.168.56.117    academia10.test

### 3) Probar en PowerShell

```powershell
curl.exe -I -k https://projectenexus10.test
curl.exe -I -k https://academia10.test
```

### 4) Probar en navegador

*   `https://projectenexus10.test`
*   `https://projectenexus10.test/no-existe` → **404 personalizado**
*   DevTools → Network → **Protocol = h2**

> **Texto para la guía:**\
> “En esta extensión se configuró una red Host‑Only entre dos VMs (Ubuntu Server y Windows 11). El cliente Windows resolvió los dominios locales mediante su archivo `hosts` y accedió por HTTPS/HTTP2 al servidor, validando la redirección y el 404 personalizado. Esta extensión simula un entorno real cliente‑servidor para pruebas de infraestructura.”

***

## 📎 Apéndice — Archivos de configuración finales

**`/etc/apache2/sites-available/projectenexus10.test.conf` (HTTP)**

```apache
<VirtualHost *:80>
    ServerName projectenexus10.test
    DocumentRoot /var/www/projectenexus10.test/public_html
    <Directory /var/www/projectenexus10.test/public_html>
        AllowOverride All
        Require all granted
    </Directory>
    ErrorDocument 404 /errors/404.html
    Redirect "/" "https://projectenexus10.test/"
    ErrorLog ${APACHE_LOG_DIR}/projectenexus10_error.log
    CustomLog ${APACHE_LOG_DIR}/projectenexus10_access.log combined
</VirtualHost>
```

**`/etc/apache2/sites-available/academia10.test.conf` (HTTP)**

```apache
<VirtualHost *:80>
    ServerName academia10.test
    DocumentRoot /var/www/academia10.test/public_html
    <Directory /var/www/academia10.test/public_html>
        AllowOverride All
        Require all granted
    </Directory>
    Redirect "/" "https://academia10.test/"
    ErrorLog ${APACHE_LOG_DIR}/academia10_error.log
    CustomLog ${APACHE_LOG_DIR}/academia10_access.log combined
</VirtualHost>
```

**`/etc/apache2/sites-available/projectenexus10-ssl.conf` (HTTPS)**

```apache
<IfModule mod_ssl.c>
<VirtualHost *:443>
    ServerName projectenexus10.test
    DocumentRoot /var/www/projectenexus10.test/public_html
    Protocols h2 h2c http/1.1
    <Directory /var/www/projectenexus10.test/public_html>
        AllowOverride All
        Require all granted
    </Directory>
    ErrorDocument 404 /errors/404.html
    SSLEngine on
    SSLCertificateFile /etc/ssl/certs/apache-selfsigned.crt
    SSLCertificateKeyFile /etc/ssl/private/apache-selfsigned.key
    ErrorLog ${APACHE_LOG_DIR}/projectenexus10_ssl_error.log
    CustomLog ${APACHE_LOG_DIR}/projectenexus10_ssl_access.log combined
</VirtualHost>
</IfModule>
```

**`/etc/apache2/sites-available/academia10-ssl.conf` (HTTPS)**

```apache
<IfModule mod_ssl.c>
<VirtualHost *:443>
    ServerName academia10.test
    DocumentRoot /var/www/academia10.test/public_html
    Protocols h2 h2c http/1.1
    <Directory /var/www/academia10.test/public_html>
        AllowOverride All
        Require all granted
    </Directory>
    SSLEngine on
    SSLCertificateFile /etc/ssl/certs/apache-selfsigned.crt
    SSLCertificateKeyFile /etc/ssl/private/apache-selfsigned.key
    ErrorLog ${APACHE_LOG_DIR}/academia10_ssl_error.log
    CustomLog ${APACHE_LOG_DIR}/academia10_ssl_access.log combined
</VirtualHost>
</IfModule>
```

***

##  Cierre


*   **HTTP/2** verificado con `curl` (cabecera `HTTP/2 200`).
*   **SSL** con **SAN** y **redirección** a HTTPS.
*   **404 personalizado** funcionando también en HTTPS.
*   **EXTRA** validado desde cliente Windows en red Host‑Only.
