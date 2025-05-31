##############################################
#  Instalación y Configuración de Servicios  #
#       DNS, Web, DHCP, FTP y Correo        #
#                (Ubuntu)                   #
##############################################

# --------------------------------------------------
# 1. DNS (Bind9)
# --------------------------------------------------

sudo apt update
sudo apt install -y bind9 bind9utils bind9-doc dnsutils

# Configurar named.conf.options
sudo tee /etc/bind/named.conf.options > /dev/null << 'EOF'
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
EOF

# Configurar named.conf.local
sudo tee /etc/bind/named.conf.local > /dev/null << 'EOF'
zone "apple.tm" {
    type master;
    file "/etc/bind/db.redes";
};

zone "22.26.172.in-addr.arpa" {
    type master;
    file "/etc/bind/db.22.26.172";
};
EOF

# Crear archivo de zona directa (db.redes)
sudo tee /etc/bind/db.redes > /dev/null << 'EOF'
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
EOF

# Crear archivo de zona inversa (db.22.26.172)
sudo tee /etc/bind/db.22.26.172 > /dev/null << 'EOF'
; Zona inversa para 172.26.22.0/24
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
EOF

# Recargar Bind9
sudo systemctl reload bind9


# --------------------------------------------------
# 2. Servidor Web (Apache2)
# --------------------------------------------------

sudo apt install -y apache2

# Permitir tráfico HTTP/HTTPS en UFW
sudo ufw allow 'Apache Full'

# Crear directorio para sitio web y asignar permisos
sudo mkdir -p /var/www/pagina_web/html
sudo chown -R $USER:$USER /var/www/pagina_web/
sudo chmod -R 755 /var/www/pagina_web/

# Configurar VirtualHost
sudo tee /etc/apache2/sites-available/pagina_web.conf > /dev/null << 'EOF'
<VirtualHost *:80>
    ServerAdmin admin@apple.tm
    ServerName apple.tm
    ServerAlias www.apple.tm
    DocumentRoot /var/www/pagina_web/html
    DirectoryIndex index.html
</VirtualHost>
EOF

# Habilitar sitio y deshabilitar el default
sudo a2ensite pagina_web.conf
sudo a2dissite 000-default.conf

# Probar configuración y recargar Apache
sudo apache2ctl configtest
sudo systemctl reload apache2


# --------------------------------------------------
# 3. DHCP (isc-dhcp-server)
# --------------------------------------------------

sudo apt install -y isc-dhcp-server

# Indicar interfaz de red (ejemplo: ens33)
sudo sed -i 's/^INTERFACESv4=.*/INTERFACESv4="ens33"/' /etc/default/isc-dhcp-server

# Configurar dhcpd.conf
sudo tee /etc/dhcp/dhcpd.conf > /dev/null << 'EOF'
authoritative;

subnet 172.26.22.0 netmask 255.255.255.0 {
    range 172.26.22.110 172.26.22.120;
    option routers 172.26.22.1;
    option domain-name-servers 172.26.22.102;
    option domain-name "apple.tm";
}
EOF

# Reiniciar servicio DHCP
sudo systemctl restart isc-dhcp-server


# --------------------------------------------------
# 4. FTP (vsftpd)
# --------------------------------------------------

sudo apt update
sudo apt install -y vsftpd

# Habilitar vsftpd al inicio y reiniciar
sudo systemctl enable vsftpd
sudo systemctl restart vsftpd

# Permitir puertos FTP en UFW
sudo ufw allow 21/tcp
sudo ufw allow 10000:10100/tcp
sudo ufw reload

# Crear respaldo de configuración y editar vsftpd.conf
sudo cp /etc/vsftpd.conf /etc/vsftpd.conf.bak

sudo tee /etc/vsftpd.conf > /dev/null << 'EOF'
listen=YES
anonymous_enable=NO
local_enable=YES
write_enable=YES
chroot_local_user=YES
allow_writeable_chroot=YES
pasv_enable=YES
pasv_min_port=10000
pasv_max_port=10100
userlist_enable=YES
userlist_file=/etc/vsftpd.userlist
userlist_deny=NO
EOF

# Crear lista de usuarios permitidos en vsftpd
echo "sumaran" | sudo tee /etc/vsftpd.userlist

# Crear usuario FTP y asignar permisos
sudo useradd -m sumaran
sudo passwd sumaran

sudo chown sumaran:sumaran /home/sumaran
sudo chmod 755 /home/sumaran

echo "Archivo de prueba FTP" | sudo tee /home/sumaran/prueba.txt
sudo chown sumaran:sumaran /home/sumaran/prueba.txt
sudo chmod 644 /home/sumaran/prueba.txt

# Reiniciar servicio vsftpd para aplicar cambios
sudo systemctl restart vsftpd


# --------------------------------------------------
# 5. Servidor de Correo (Postfix + Dovecot)
# --------------------------------------------------

sudo apt install -y postfix mailutils dovecot-core dovecot-imapd dovecot-pop3d

# Configurar Postfix
sudo postconf -e "myhostname = mail.apple.tm"
sudo postconf -e "mydomain = apple.tm"
sudo postconf -e "myorigin = /etc/mailname"
sudo postconf -e "mydestination = \$myhostname, localhost.\$mydomain, localhost, \$mydomain"
sudo postconf -e "relayhost ="
sudo postconf -e "mynetworks = 127.0.0.0/8, 172.26.22.0/24"
sudo postconf -e "mailbox_command ="
sudo postconf -e "mailbox_size_limit = 0"
sudo postconf -e "recipient_delimiter = +"
sudo postconf -e "inet_interfaces = all"
sudo postconf -e "inet_protocols = ipv4"

# Configurar Dovecot: 10-auth.conf
sudo tee /etc/dovecot/conf.d/10-auth.conf > /dev/null << 'EOF'
disable_plaintext_auth = no
auth_mechanisms = plain login
EOF

# Configurar Dovecot: 10-mail.conf
sudo tee /etc/dovecot/conf.d/10-mail.conf > /dev/null << 'EOF'
mail_location = maildir:~/Maildir
EOF

# Configurar Dovecot: 10-ssl.conf
sudo tee /etc/dovecot/conf.d/10-ssl.conf > /dev/null << 'EOF'
ssl = no
EOF

# Reiniciar servicios de correo
sudo systemctl restart postfix
sudo systemctl restart dovecot

# Enviar correo de prueba local
echo "Subject: Correo Maildir 1" | sendmail sumaran@mail.apple.tm
