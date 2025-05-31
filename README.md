# DNS - Bind9
sudo apt update
sudo apt install -y bind9 bind9utils bind9-doc dnsutils

cd /etc/bind/
sudo nano named.conf.options
# Reemplazar con:
cat << EOF | sudo tee named.conf.options
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

sudo nano named.conf.local
# Reemplazar con:
cat << EOF | sudo tee named.conf.local
zone "apple.tm" {
    type master;
    file "/etc/bind/db.redes";
};

zone "22.26.172.in-addr.arpa" {
    type master;
    file "/etc/bind/db.22.26.172";
};
EOF

sudo nano /etc/bind/db.redes
# Reemplazar con:
cat << EOF | sudo tee /etc/bind/db.redes
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

sudo nano /etc/bind/db.22.26.172
# Reemplazar con:
cat << EOF | sudo tee /etc/bind/db.22.26.172
; Zona inversa
\$TTL    604800
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

sudo systemctl reload bind9

# Servidor Web - Apache2
sudo apt install -y apache2
sudo ufw allow 'Apache Full'

sudo mkdir -p /var/www/pagina_web/html
sudo chown -R $USER:$USER /var/www/pagina_web/
sudo chmod -R 755 /var/www/pagina_web/

sudo nano /etc/apache2/sites-available/pagina_web.conf
# Reemplazar con:
cat << EOF | sudo tee /etc/apache2/sites-available/pagina_web.conf
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

# DHCP - isc-dhcp-server
sudo apt install -y isc-dhcp-server
sudo nano /etc/default/isc-dhcp-server
# Cambiar INTERFACESv4="ens33" (según interfaz)

sudo nano /etc/dhcp/dhcpd.conf
# Reemplazar con:
cat << EOF | sudo tee /etc/dhcp/dhcpd.conf
authoritative;

subnet 172.26.22.0 netmask 255.255.255.0 {
  range 172.26.22.110 172.26.22.120;
  option routers 172.26.22.1;
  option domain-name-servers 172.26.22.102;
  option domain-name "apple.tm";
}
EOF

sudo systemctl restart isc-dhcp-server

# FTP - vsftpd
sudo apt update
sudo apt install -y vsftpd
sudo systemctl enable vsftpd
sudo systemctl restart vsftpd

sudo ufw allow 21/tcp
sudo ufw allow 10000:10100/tcp
sudo ufw reload

sudo cp /etc/vsftpd.conf /etc/vsftpd.conf.bak
sudo nano /etc/vsftpd.conf
# Reemplazar con:
cat << EOF | sudo tee /etc/vsftpd.conf
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

echo "sumaran" | sudo tee /etc/vsftpd.userlist

sudo useradd -m sumaran
sudo passwd sumaran

sudo chown sumaran:sumaran /home/sumaran
sudo chmod 755 /home/sumaran

echo "Archivo de prueba FTP" | sudo tee /home/sumaran/prueba.txt
sudo chown sumaran:sumaran /home/sumaran/prueba.txt
sudo chmod 644 /home/sumaran/prueba.txt

sudo systemctl restart vsftpd

# Correo - Postfix + Dovecot
sudo apt install -y postfix mailutils dovecot-core dovecot-imapd dovecot-pop3d

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

sudo nano /etc/dovecot/conf.d/10-auth.conf
# Cambiar o agregar:
# disable_plaintext_auth = no
# auth_mechanisms = plain login

sudo nano /etc/dovecot/conf.d/10-mail.conf
# Cambiar o agregar:
# mail_location = maildir:~/Maildir

sudo nano /etc/dovecot/conf.d/10-ssl.conf
# Cambiar o agregar:
# ssl = no

sudo systemctl restart postfix
sudo systemctl restart dovecot

echo "Subject: Correo Maildir 1" | sendmail sumaran@mail.apple.tm
