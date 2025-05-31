# Instalación y Configuración de Servicios de Red

Este repositorio contiene la configuración completa para los servicios DNS, Web, DHCP, FTP y Correo en un servidor Linux.

## Servicios Configurados

- **DNS** (BIND9)
- **Servidor Web** (Apache2)
- **DHCP** (ISC-DHCP-Server)
- **FTP** (vsftpd)
- **Correo** (Postfix + Dovecot)

---

## 1. Configuración DNS (BIND9)

### Configuración principal

```bash
cd /etc/bind/
nano named.conf.options
```

**Archivo: `/etc/bind/named.conf.options`**
```bash
options {
    directory "/var/cache/bind";

    forwarders {
        8.8.8.8;
        8.8.4.4;
    };

    dnssec-validation auto;

    listen-on { any; };
    listen-on-v6 { any; };
    allow-query { any; };
};
```

### Configuración de zonas

```bash
sudo nano named.conf.local
```

**Archivo: `/etc/bind/named.conf.local`**
```bash
zone "apple.tm" {
    type master;
    file "/etc/bind/db.redes";
};

zone "22.26.172.in-addr.arpa" {
    type master;
    file "/etc/bind/db.22.26.172";
};
```

### Zona directa

```bash
sudo nano db.redes
```

**Archivo: `/etc/bind/db.redes`**
```bash
$TTL 604800
@   IN  SOA apple.tm. admin.apple.tm. (
            2025052901 ; Serial
            10h        ; Refresh
            15m        ; Retry
            48h        ; Expire
            604800     ; Negative Cache TTL
)

@       IN  NS      apple.tm.
@       IN  A       172.26.22.102
mail    IN  A       172.26.22.102
www     IN  CNAME   apple.tm.
@       IN  MX 10   apple.tm.
cisco   IN  A       172.26.22.102
```

### Zona inversa

```bash
sudo nano db.22.26.172
```

**Archivo: `/etc/bind/db.22.26.172`**
```bash
$TTL    604800
@       IN  SOA     apple.tm. admin.apple.tm. (
                        2025052901 ; Serial
                        10h        ; Refresh
                        15m        ; Retry
                        48h        ; Expire
                        604800     ; Negative Cache TTL
)

@       IN  NS      apple.tm.
102     IN  PTR     apple.tm.
```

### Reiniciar servicio DNS

```bash
sudo systemctl reload bind9
```

---

## 2. Configuración Servidor Web (Apache2)

### Configuración del firewall

```bash
sudo ufw status
sudo ufw app list
sudo ufw allow 'Apache Full'
```

### Configuración de permisos

```bash
sudo chown -R $USER:$USER /var/www/pagina_web/
sudo chmod -R 755 /var/www/pagina_web/
```

### Configuración del Virtual Host

```bash
cd /etc/apache2/sites-available/
nano pagina_web.conf
```

**Archivo: `/etc/apache2/sites-available/pagina_web.conf`**
```apache
<VirtualHost *:80>
    ServerAdmin admin@apple.tm
    ServerName apple.tm
    ServerAlias www.apple.tm
    DocumentRoot /var/www/pagina_web/html
    DirectoryIndex index.html
</VirtualHost>
```

### Habilitar sitio

```bash
sudo a2ensite pagina_web
sudo a2dissite 000-default.conf
sudo apache2ctl configtest
sudo systemctl reload apache2
```

**URL de prueba:** `http://www.apple.tm/`

---

## 3. Configuración DHCP

### Instalación

```bash
sudo apt install isc-dhcp-server
```

### Configuración de interfaz

```bash
sudo nano /etc/default/isc-dhcp-server
```

**Archivo: `/etc/default/isc-dhcp-server`**
```bash
INTERFACESv4="ens33"
```

### Configuración del pool DHCP

```bash
sudo nano /etc/dhcp/dhcpd.conf
```

**Archivo: `/etc/dhcp/dhcpd.conf`**
```bash
authoritative;

subnet 172.26.22.0 netmask 255.255.255.0 {
  range 172.26.22.110 172.26.22.120;
  option routers rtr-22-0-1.apple.tm, rtr-22-0-2.apple.tm;
}
```

### Reiniciar servicio

```bash
sudo systemctl restart isc-dhcp-server
```

---

## 4. Configuración FTP (vsftpd)

### Instalación

```bash
sudo apt update
sudo apt install vsftpd
sudo systemctl status vsftpd
sudo systemctl restart vsftpd
sudo systemctl enable vsftpd
```

### Configuración del firewall

```bash
sudo ufw allow 21/tcp
sudo ufw allow 10000:10100/tcp  # para puertos pasivos
sudo ufw reload
```

### Backup de configuración

```bash
sudo cp /etc/vsftpd.conf /etc/vsftpd.conf.bak
```

### Lista de usuarios FTP

```bash
sudo nano /etc/vsftpd.userlist
```

**Archivo: `/etc/vsftpd.userlist`**
```
sumaran
```

### Configuración principal

```bash
sudo nano /etc/vsftpd.conf
```

**Archivo: `/etc/vsftpd.conf`**
```bash
listen=YES
anonymous_enable=NO
local_enable=YES
write_enable=YES
chroot_local_user=YES
allow_writeable_chroot=YES
pasv_enable=YES
pasv_min_port=10000
pasv_max_port=10100
```

### Configuración de permisos de usuario

```bash
sudo chown sumaran:sumaran /home/sumaran
sudo chmod 755 /home/sumaran

echo "Archivo de prueba FTP" > /home/sumaran/prueba.txt
sudo chown sumaran:sumaran /home/sumaran/prueba.txt
sudo chmod 644 /home/sumaran/prueba.txt
```

---

## 5. Configuración Servidor de Correo

### Configuración DNS

```bash
nano /etc/resolv.conf
```

### Instalación Postfix

```bash
sudo apt install postfix
```

### Configuración Postfix

```bash
nano /etc/postfix/main.cf
```

### Instalación de utilidades de correo

```bash
sudo apt install mailutils
sudo apt install dovecot-imapd dovecot-pop3d
apt-get install dovecot-core dovecot-imapd
```

### Configuración Dovecot

```bash
nano /etc/dovecot/conf.d/10-ssl.conf
nano /etc/dovecot/conf.d/10-auth.conf 
nano /etc/dovecot/conf.d/10-mail.conf
```

### Reiniciar servicios

```bash
systemctl restart postfix
```

### Prueba de envío de correo

```bash
echo "Subject: Correo Maildir 1" | sendmail luis@mail.apple.tm
```

---

## Cuentas de Correo Configuradas

- `sumaran@mail.apple.tm`
- `luis@mail.apple.tm`

---

## Referencias

- [Configuración Web Apache2](https://www.youtube.com/watch?v=aTb9JeKp5TA&t=48s&ab_channel=SebaselInge)
- [Configuración DHCP](https://www.youtube.com/watch?v=JCeILZwKUOI&ab_channel=soujux)
- [Configuración FTP](https://www.youtube.com/watch?v=2c89x6S_ZKk&ab_channel=JoseCesarSa%C3%B1ezSalce)

---

## Dominio Principal

**Dominio:** `apple.tm`  
**IP del Servidor:** `172.26.22.102`  
**Red:** `172.26.22.0/24`