# ==========================================
# Instalación y Configuración de Servicios
# DNS, Web, DHCP, FTP, y Correo en Ubuntu
# ==========================================

# --------- 1. DNS (Bind9) ----------------
sudo apt update
sudo apt install -y bind9 bind9utils bind9-doc dnsutils

# Configurar named.conf.options
cat <<EOF | sudo tee /etc/bind/named.conf.options
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

# Configurar named.conf.local con zonas
cat <<EOF | sudo tee /etc/bind/named.conf.local
zone "apple.tm" {
    type master;
    file "/etc/bind/db.apple.tm";
};

zone "22.26.172.in-addr.arpa" {
    type master;
    file "/etc/bind/db.22.26.172";
};
EOF

# Crear archivo de zona directa
cat <<EOF | sudo tee /etc/bind/db.apple.tm
\$TTL 604800
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

# Crear archivo de zona inversa
cat <<EOF | sudo tee /etc/bind/db.22.26.172
\$TTL 604800
@   IN  SOA apple.tm. admin.apple.tm. (
            2025052901 ; Serial
            10h        ; Refresh
            15m        ; Retry
            48h        ; Expire
            604800     ; Negative Cache TTL
)
@       IN  NS      apple.tm.
102     IN  PTR     apple.tm.
EOF

sudo systemctl reload bind9

# --------- 2. Servidor Web (Apache2) ----------------
sudo apt install -y apache2
sudo ufw allow 'Apache Full'

# Cambiar permisos a carpeta web (ajustar $USER y ruta según sea necesario)
sudo chown -R $USER:$USER /var/www/pagina_web/
sudo chmod -R 755 /var/www/pagina_web/

# Crear archivo de configuración del sitio
cat <<EOF | sudo tee /etc/apache2/sites-available/pagina_web.conf
<VirtualHost *:80>
    ServerAdmin admin@apple.tm
    ServerName apple.tm
    ServerAlias www.apple.tm
    DocumentRoot /var/www/pagina_web/html
    DirectoryIndex index.html
</VirtualHost>
EOF

sudo a2ensite pagina_web.conf
sudo a2dissite 000-default.conf
sudo apache2ctl configtest
sudo systemctl reload apache2

# --------- 3. DHCP Server (isc-dhcp-server) ----------------
sudo apt install -y isc-dhcp-server

# Configurar interfaz en /etc/default/isc-dhcp-server
sudo sed -i 's/^INTERFACESv4=""/INTERFACESv4="ens33"/' /etc/default/isc-dhcp-server

# Configurar DHCP en /etc/dhcp/dhcpd.conf
cat <<EOF | sudo tee /etc/dhcp/dhcpd.conf
authoritative;

subnet 172.26.22.0 netmask 255.255.255.0 {
  range 172.26.22.110 172.26.22.120;
  option routers rtr-22-0-1.apple.tm, rtr-22-0-2.apple.tm;
}
EOF

sudo systemctl restart isc-dhcp-server

# --------- 4. FTP Server (vsftpd) ----------------
sudo apt update
sudo apt install -y vsftpd
sudo systemctl enable --now vsftpd

sudo ufw allow 21/tcp
sudo ufw allow 10000:10100/tcp
sudo ufw reload

# Respaldar configuración original
sudo cp /etc/vsftpd.conf /etc/vsftpd.conf.bak

# Configurar vsftpd
cat <<EOF | sudo tee /etc/vsftpd.conf
listen=YES
anonymous_enable=NO
local_enable=YES
write_enable=YES
chroot_local_user=YES
allow_writeable_chroot=YES
pasv_enable=YES
pasv_min_port=10000
pasv_max_port=10100
EOF

# Ajustar permisos usuario FTP (reemplazar 'sumaran' por usuario real)
sudo chown sumaran:sumaran /home/sumaran
sudo chmod 755 /home/sumaran

echo "Archivo de prueba FTP" | sudo tee /home/sumaran/prueba.txt
sudo chown sumaran:sumaran /home/sumaran/prueba.txt
sudo chmod 644 /home/sumaran/prueba.txt

# --------- 5. Servidor de Correo (Postfix + Dovecot) ----------------
sudo apt install -y postfix mailutils dovecot-core dovecot-imapd dovecot-pop3d

# (Opcional) Editar configuración de Postfix en /etc/postfix/main.cf según necesidades
# (Opcional) Configurar SSL y autenticación en Dovecot editando:
# /etc/dovecot/conf.d/10-ssl.conf
# /etc/dovecot/conf.d/10-auth.conf
# /etc/dovecot/conf.d/10-mail.conf

sudo systemctl restart postfix dovecot

# Test envío de correo (reemplazar destinatario)
echo "Subject: Correo Maildir 1" | sendmail luis@mail.apple.tm

# Fin
