# 🌐 Configuración Completa de Servicios de Red

> **Guía completa para la instalación y configuración de servicios DNS, Web, DHCP, FTP y Correo en Ubuntu/Debian**

[![Ubuntu](https://img.shields.io/badge/Ubuntu-20.04%2B-orange.svg)](https://ubuntu.com/)
[![Debian](https://img.shields.io/badge/Debian-10%2B-red.svg)](https://debian.org/)
[![License](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

## 📋 Tabla de Contenidos

- [🔧 Servicios Incluidos](#-servicios-incluidos)
- [🏗️ Arquitectura de Red](#️-arquitectura-de-red)
- [🌍 Servidor DNS (BIND9)](#-servidor-dns-bind9)
- [🌐 Servidor Web (Apache2)](#-servidor-web-apache2)
- [📡 Servidor DHCP](#-servidor-dhcp)
- [📁 Servidor FTP (vsftpd)](#-servidor-ftp-vsftpd)
- [📧 Servidor de Correo](#-servidor-de-correo)
- [✅ Verificación y Pruebas](#-verificación-y-pruebas)
- [🔗 Referencias](#-referencias)

---

## 🔧 Servicios Incluidos

| Servicio | Puerto | Protocolo | Estado |
|----------|--------|-----------|--------|
| DNS (BIND9) | 53 | UDP/TCP | ✅ Configurado |
| Web (Apache2) | 80 | HTTP | ✅ Configurado |
| DHCP | 67/68 | UDP | ✅ Configurado |
| FTP (vsftpd) | 21, 10000-10100 | TCP | ✅ Configurado |
| Mail (Postfix/Dovecot) | 25, 143, 110 | TCP | ✅ Configurado |

---

## 🏗️ Arquitectura de Red

```
Dominio: apple.tm
Servidor: 172.26.22.102
Red: 172.26.22.0/24
DHCP Pool: 172.26.22.110 - 172.26.22.120
```

---

## 🌍 Servidor DNS (BIND9)

### 📦 Instalación

```bash
# Actualizar repositorios
sudo apt update

# Instalar BIND9 y utilidades
sudo apt install bind9 bind9utils bind9-doc dnsutils -y

# Verificar instalación
sudo systemctl status bind9
```

### ⚙️ Configuración Principal

```bash
cd /etc/bind/
sudo nano named.conf.options
```

**📄 Archivo: `/etc/bind/named.conf.options`**
```bash
options {
    directory "/var/cache/bind";

    // Configuración de reenviadores DNS
    forwarders {
        8.8.8.8;        // Google DNS
        8.8.4.4;        // Google DNS alternativo
    };

    // Validación DNSSEC
    dnssec-validation auto;

    // Configuración de escucha
    listen-on { any; };
    listen-on-v6 { any; };
    
    // Permitir consultas desde cualquier dirección
    allow-query { any; };
};
```

### 🌐 Configuración de Zonas

```bash
sudo nano /etc/bind/named.conf.local
```

**📄 Archivo: `/etc/bind/named.conf.local`**
```bash
// Zona directa para apple.tm
zone "apple.tm" {
    type master;
    file "/etc/bind/db.redes";
};

// Zona inversa para la red 172.26.22.0/24
zone "22.26.172.in-addr.arpa" {
    type master;
    file "/etc/bind/db.22.26.172";
};
```

### 📝 Zona Directa

```bash
sudo nano /etc/bind/db.redes
```

**📄 Archivo: `/etc/bind/db.redes`**
```bash
$TTL 604800
@   IN  SOA apple.tm. admin.apple.tm. (
            2025052901 ; Serial (YYYYMMDDNN)
            10h        ; Refresh
            15m        ; Retry
            48h        ; Expire
            604800     ; Negative Cache TTL
)

; Servidor de nombres
@       IN  NS      apple.tm.

; Registro A principal
@       IN  A       172.26.22.102

; Servicios
mail    IN  A       172.26.22.102
www     IN  CNAME   apple.tm.
cisco   IN  A       172.26.22.102

; Registro MX para correo
@       IN  MX 10   apple.tm.
```

### 🔄 Zona Inversa

```bash
sudo nano /etc/bind/db.22.26.172
```

**📄 Archivo: `/etc/bind/db.22.26.172`**
```bash
$TTL    604800
@       IN  SOA     apple.tm. admin.apple.tm. (
                        2025052901 ; Serial
                        10h        ; Refresh
                        15m        ; Retry
                        48h        ; Expire
                        604800     ; Negative Cache TTL
)

; Servidor de nombres
@       IN  NS      apple.tm.

; Resolución inversa
102     IN  PTR     apple.tm.
```

### 🚀 Activación del Servicio

```bash
# Verificar configuración
sudo named-checkconf
sudo named-checkzone apple.tm /etc/bind/db.redes
sudo named-checkzone 22.26.172.in-addr.arpa /etc/bind/db.22.26.172

# Reiniciar y habilitar servicio
sudo systemctl reload bind9
sudo systemctl enable bind9
sudo systemctl status bind9
```

---

## 🌐 Servidor Web (Apache2)

### 📦 Instalación

```bash
# Instalar Apache2
sudo apt update
sudo apt install apache2 -y

# Iniciar y habilitar servicio
sudo systemctl start apache2
sudo systemctl enable apache2
sudo systemctl status apache2
```

### 🔥 Configuración del Firewall

```bash
# Verificar estado del firewall
sudo ufw status

# Ver aplicaciones disponibles
sudo ufw app list

# Permitir tráfico HTTP y HTTPS
sudo ufw allow 'Apache Full'
```

### 📁 Estructura del Sitio Web

```bash
# Crear directorio para el sitio
sudo mkdir -p /var/www/pagina_web/html

# Crear página de prueba
echo "<h1>¡Bienvenido a apple.tm!</h1>" | sudo tee /var/www/pagina_web/html/index.html
```

### 👤 Configuración de Permisos

```bash
# Asignar propietario
sudo chown -R $USER:$USER /var/www/pagina_web/

# Establecer permisos
sudo chmod -R 755 /var/www/pagina_web/
```

### 🏠 Virtual Host

```bash
cd /etc/apache2/sites-available/
sudo nano pagina_web.conf
```

**📄 Archivo: `/etc/apache2/sites-available/pagina_web.conf`**
```apache
<VirtualHost *:80>
    # Información del administrador
    ServerAdmin admin@apple.tm
    
    # Configuración del dominio
    ServerName apple.tm
    ServerAlias www.apple.tm
    
    # Directorio raíz del sitio
    DocumentRoot /var/www/pagina_web/html
    
    # Archivo índice por defecto
    DirectoryIndex index.html
    
    # Logs del sitio
    ErrorLog ${APACHE_LOG_DIR}/apple_error.log
    CustomLog ${APACHE_LOG_DIR}/apple_access.log combined
</VirtualHost>
```

### 🔄 Activación del Sitio

```bash
# Habilitar nuevo sitio
sudo a2ensite pagina_web

# Deshabilitar sitio por defecto
sudo a2dissite 000-default.conf

# Verificar configuración
sudo apache2ctl configtest

# Recargar Apache
sudo systemctl reload apache2
```

**🌐 Prueba:** `http://www.apple.tm/` o `http://apple.tm/`

---

## 📡 Servidor DHCP

### 📦 Instalación

```bash
# Instalar servidor DHCP
sudo apt update
sudo apt install isc-dhcp-server -y
```

### 🔌 Configuración de Interfaz

```bash
sudo nano /etc/default/isc-dhcp-server
```

**📄 Archivo: `/etc/default/isc-dhcp-server`**
```bash
# Interfaz donde escuchará el servidor DHCP
INTERFACESv4="ens33"
INTERFACESv6=""
```

### 🌐 Pool de Direcciones IP

```bash
sudo nano /etc/dhcp/dhcpd.conf
```

**📄 Archivo: `/etc/dhcp/dhcpd.conf`**
```bash
# Declarar como servidor autoritativo
authoritative;

# Configuración global
default-lease-time 600;
max-lease-time 7200;

# Subred y pool de IPs
subnet 172.26.22.0 netmask 255.255.255.0 {
    # Rango de IPs a asignar
    range 172.26.22.110 172.26.22.120;
    
    # Puerta de enlace
    option routers 172.26.22.1;
    
    # Servidores DNS
    option domain-name-servers 172.26.22.102;
    
    # Dominio de búsqueda
    option domain-name "apple.tm";
    
    # Servidor de tiempo
    option broadcast-address 172.26.22.255;
}
```

### 🚀 Activación del Servicio

```bash
# Verificar configuración
sudo dhcpd -t -cf /etc/dhcp/dhcpd.conf

# Reiniciar y habilitar servicio
sudo systemctl restart isc-dhcp-server
sudo systemctl enable isc-dhcp-server
sudo systemctl status isc-dhcp-server

# Ver logs del DHCP
sudo journalctl -u isc-dhcp-server -f
```

---

## 📁 Servidor FTP (vsftpd)

### 📦 Instalación

```bash
# Instalar vsftpd
sudo apt update
sudo apt install vsftpd -y

# Verificar instalación
sudo systemctl status vsftpd
sudo systemctl enable vsftpd
```

### 🔥 Configuración del Firewall

```bash
# Permitir puerto FTP
sudo ufw allow 21/tcp

# Permitir rango de puertos pasivos
sudo ufw allow 10000:10100/tcp

# Recargar firewall
sudo ufw reload
sudo ufw status
```

### 💾 Respaldo de Configuración

```bash
# Crear copia de seguridad
sudo cp /etc/vsftpd.conf /etc/vsftpd.conf.backup
```

### 👥 Usuarios FTP

```bash
# Crear lista de usuarios permitidos
sudo nano /etc/vsftpd.userlist
```

**📄 Archivo: `/etc/vsftpd.userlist`**
```
sumaran
```

### ⚙️ Configuración Principal

```bash
sudo nano /etc/vsftpd.conf
```

**📄 Archivo: `/etc/vsftpd.conf`**
```bash
# Configuración básica
listen=YES
listen_ipv6=NO

# Deshabilitar acceso anónimo
anonymous_enable=NO

# Habilitar usuarios locales
local_enable=YES

# Permitir escritura
write_enable=YES

# Enjaular usuarios en su directorio home
chroot_local_user=YES
allow_writeable_chroot=YES

# Configuración de modo pasivo
pasv_enable=YES
pasv_min_port=10000
pasv_max_port=10100

# Configuración de usuarios
userlist_enable=YES
userlist_file=/etc/vsftpd.userlist
userlist_deny=NO

# Configuración de logs
xferlog_enable=YES
xferlog_file=/var/log/vsftpd.log
```

### 👤 Configuración de Usuario

```bash
# Crear usuario si no existe
sudo useradd -m sumaran
sudo passwd sumaran

# Configurar permisos del directorio home
sudo chown sumaran:sumaran /home/sumaran
sudo chmod 755 /home/sumaran

# Crear archivo de prueba
echo "¡Bienvenido al servidor FTP de apple.tm!" | sudo tee /home/sumaran/bienvenida.txt
sudo chown sumaran:sumaran /home/sumaran/bienvenida.txt
sudo chmod 644 /home/sumaran/bienvenida.txt
```

### 🚀 Activación del Servicio

```bash
# Reiniciar servicio
sudo systemctl restart vsftpd
sudo systemctl status vsftpd

# Verificar configuración
sudo vsftpd -olisten=NO /etc/vsftpd.conf
```

**🔌 Prueba de conexión:**
```bash
ftp 172.26.22.102
# Usuario: sumaran
# Contraseña: [la que configuraste]
```

---

## 📧 Servidor de Correo

### 🌐 Configuración DNS Previa

```bash
# Configurar resolución DNS local
echo "nameserver 172.26.22.102" | sudo tee /etc/resolv.conf
```

### 📦 Instalación de Postfix

```bash
# Instalar Postfix (seleccionar "Internet Site")
sudo apt update
sudo apt install postfix -y

# Instalar utilidades de correo
sudo apt install mailutils -y
```

### ⚙️ Configuración de Postfix

```bash
sudo nano /etc/postfix/main.cf
```

**📄 Configuraciones importantes en `/etc/postfix/main.cf`:**
```bash
# Nombre del sistema de correo
myhostname = mail.apple.tm

# Dominio de origen
mydomain = apple.tm

# Origen del correo
myorigin = $mydomain

# Interfaces de red
inet_interfaces = all

# Protocolos
inet_protocols = ipv4

# Dominios de destino
mydestination = $myhostname, localhost.$mydomain, localhost, $mydomain

# Formato de buzón
home_mailbox = Maildir/
```

### 📦 Instalación de Dovecot

```bash
# Instalar componentes de Dovecot
sudo apt install dovecot-core dovecot-imapd dovecot-pop3d -y
```

### ⚙️ Configuración de Dovecot

```bash
# Configuración SSL
sudo nano /etc/dovecot/conf.d/10-ssl.conf
```

```bash
# Configuración de autenticación
sudo nano /etc/dovecot/conf.d/10-auth.conf
```

```bash
# Configuración de correo
sudo nano /etc/dovecot/conf.d/10-mail.conf
```

**📄 En `/etc/dovecot/conf.d/10-mail.conf`:**
```bash
mail_location = maildir:~/Maildir
```

### 👥 Usuarios de Correo

```bash
# Crear usuarios para correo electrónico
sudo useradd -m -s /bin/bash luis
sudo passwd luis

sudo useradd -m -s /bin/bash sumaran
sudo passwd sumaran
```

### 🚀 Activación de Servicios

```bash
# Reiniciar Postfix
sudo systemctl restart postfix
sudo systemctl enable postfix

# Reiniciar Dovecot
sudo systemctl restart dovecot
sudo systemctl enable dovecot

# Verificar estado
sudo systemctl status postfix
sudo systemctl status dovecot
```

### ✉️ Prueba de Correo

```bash
# Enviar correo de prueba
echo "Este es un correo de prueba desde el servidor apple.tm" | mail -s "Prueba de Correo" luis@apple.tm

# Verificar cola de correo
mailq

# Ver logs de correo
sudo tail -f /var/log/mail.log
```

---

## ✅ Verificación y Pruebas

### 🔍 Estado de Servicios

```bash
#!/bin/bash
echo "=== ESTADO DE SERVICIOS ==="
echo "DNS (BIND9):"
sudo systemctl is-active bind9

echo "Web (Apache2):"
sudo systemctl is-active apache2

echo "DHCP:"
sudo systemctl is-active isc-dhcp-server

echo "FTP (vsftpd):"
sudo systemctl is-active vsftpd

echo "Correo (Postfix):"
sudo systemctl is-active postfix

echo "Correo (Dovecot):"
sudo systemctl is-active dovecot
```

### 🧪 Pruebas de Funcionalidad

```bash
# Prueba DNS
nslookup apple.tm 172.26.22.102
dig @172.26.22.102 apple.tm

# Prueba Web
curl -I http://apple.tm/

# Prueba de puertos
nmap -p 21,25,53,80,110,143 172.26.22.102
```

### 📊 Logs del Sistema

```bash
# Ver logs en tiempo real
sudo journalctl -f

# Logs específicos por servicio
sudo journalctl -u bind9 -f
sudo journalctl -u apache2 -f
sudo journalctl -u isc-dhcp-server -f
sudo journalctl -u vsftpd -f
sudo journalctl -u postfix -f
sudo journalctl -u dovecot -f
```

---

## 👥 Cuentas Configuradas

### 📧 Correo Electrónico
- `luis@apple.tm`
- `sumaran@apple.tm`

### 📁 FTP
- `sumaran` (con acceso completo)

---

## 🔗 Referencias

| Servicio | Tutorial | Canal |
|----------|----------|-------|
| **Apache2** | [Configuración Web](https://www.youtube.com/watch?v=aTb9JeKp5TA&t=48s) | SebaselInge |
| **DHCP** | [Configuración DHCP](https://www.youtube.com/watch?v=JCeILZwKUOI) | soujux |
| **FTP** | [Configuración FTP](https://www.youtube.com/watch?v=2c89x6S_ZKk) | José Cesar Sañez Salce |

## 📜 Licencia

Este proyecto está bajo la Licencia MIT - ver el archivo [LICENSE](LICENSE) para más detalles.

## 🤝 Contribuciones

Las contribuciones son bienvenidas. Por favor, abre un issue primero para discutir los cambios que te gustaría hacer.

---

**🎯 Desarrollado para fines educativos y de laboratorio**