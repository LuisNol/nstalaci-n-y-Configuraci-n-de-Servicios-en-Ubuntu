# CONFIGURACIÓN DE SERVICIOS: DNS, WEB, DHCP, FTP Y CORREO

## DNS (BIND9)

# Editar configuración general
cd /etc/bind/
nano named.conf.options

# Contenido de named.conf.options:
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

# Configurar zonas locales
sudo nano named.conf.local

# Agregar:
zone "apple.tm" {
    type master;
    file "/etc/bind/db.redes";
};

zone "22.26.172.in-addr.arpa" {
    type master;
    file "/etc/bind/db.22.26.172";
};

# Archivo db.redes
sudo nano /etc/bind/db.redes

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

# Archivo de zona inversa db.22.26.172.in-addr.arpa
sudo nano /etc/bind/db.22.26.172.in-addr.arpa

$TTL 604800
@       IN  SOA     apple.tm. admin.apple.tm. (
                        2025052901 ; Serial
                        10h        ; Refresh
                        15m        ; Retry
                        48h        ; Expire
                        604800     ; Negative Cache TTL
)

@       IN  NS      apple.tm.
102     IN  PTR     apple.tm.

# Recargar configuración
sudo systemctl reload bind9


## SERVICIO WEB (Apache)

sudo ufw status
sudo ufw app list
sudo ufw allow 'apache full'

sudo chown -R $USER:$USER /var/www/pagina_web/
sudo chmod -R 755 /var/www/pagina_web/

cd /etc/apache2/sites-available/

# Crear archivo de sitio virtual (ejemplo: pagina_web.conf)
<VirtualHost *:80>
    ServerAdmin admin@apple.tm
    ServerName apple.tm
    ServerAlias www.apple.tm
    DocumentRoot /var/www/pagina_web/html
    DirectoryIndex index.html
</VirtualHost>

sudo a2ensite pagina_web.conf
sudo a2dissite 000-default.conf

sudo apache2ctl configtest
sudo systemctl reload apache2

# Probar en el navegador
http://www.apple.tm/


## DHCP (isc-dhcp-server)

sudo apt install isc-dhcp-server

# Configurar interfaz en /etc/default/isc-dhcp-server
sudo nano /etc/default/isc-dhcp-server
# Modificar INTERFACESv4="ens33"

sudo nano /etc/dhcp/dhcpd.conf

authoritative;

subnet 172.26.22.0 netmask 255.255.255.0 {
  range 172.26.22.110 172.26.22.120;
  option routers rtr-22-0-1.apple.tm, rtr-22-0-2.apple.tm;
}

sudo systemctl restart isc-dhcp-server


## FTP (vsftpd)

sudo apt update
sudo apt install vsftpd

sudo systemctl status vsftpd
sudo systemctl restart vsftpd
sudo systemctl enable vsftpd

sudo ufw allow 21/tcp
sudo ufw allow 10000:10100/tcp  # para puertos pasivos
sudo ufw reload

sudo cp /etc/vsftpd.conf /etc/vsftpd.conf.bak

sudo nano /etc/vsftpd.userlist
# Añadir usuario: sumaran

sudo nano /etc/vsftpd.conf
# Configuraciones principales:
listen=YES
anonymous_enable=NO
local_enable=YES
write_enable=YES
chroot_local_user=YES
allow_writeable_chroot=YES
pasv_enable=YES
pasv_min_port=10000
pasv_max_port=10100

sudo chown sumaran:sumaran /home/sumaran
sudo chmod 755 /home/sumaran

echo "Archivo de prueba FTP" > /home/sumaran/prueba.txt
sudo chown sumaran:sumaran /home/sumaran/prueba.txt
sudo chmod 644 /home/sumaran/prueba.txt


## MAIL (Postfix y Dovecot)

sudo apt install postfix
sudo apt install mailutils
sudo apt install dovecot-core dovecot-imapd dovecot-pop3d

# Editar configuración principal postfix
sudo nano /etc/postfix/main.cf

# Configuraciones Dovecot:
sudo nano /etc/dovecot/conf.d/10-ssl.conf
sudo nano /etc/dovecot/conf.d/10-auth.conf
sudo nano /etc/dovecot/conf.d/10-mail.conf

sudo systemctl restart postfix

# Enviar correo de prueba
echo "Subject: Correo Maildir 1" | sendmail luis@mail.apple.tm
