# ansible redteam

## Protect

```
apt install ufw -y
systemctl enable ufw
systemctl start ufw
ufw enable
ufw allow <app>
```

## gophish

### Install

```
go get github.com/gophish/gophish

git clone https://github.com/gophish/gophish

cd gophish

go build
```

### Connect

```
ssh -L 3333:localhost:3333 likes-webdeliv1
```

## web deliv

### apt

```
apt install php libapache2-mod-php apache2 -y
```

### Configure Apache

```
cd /etc/apache2

# listen on 443
sed -i.bak 's/^Listen 80/\#Listen 80/g' ports.conf

# enable ssh module
a2enmod ssl

# disable directory indexing
sed -i.bak 's/Options Indexes FollowSymLinks/Options FollowSymLinks/g' apache2.conf

# ditch 80/tcp
rm sites-enabled/000-default.conf
```

### Configure Virtual Host

Drop the vhost config

```
# /etc/sites-available/somesite.conf
<IfModule mod_ssl.c>
        <VirtualHost some.site.com>
        ServerName some.site.com
                ServerAdmin webmaster@localhost

        DocumentRoot /var/www/html/
        DirectoryIndex index.php index.html

                ErrorLog ${APACHE_LOG_DIR}/web_delivery-www-error.log
                CustomLog ${APACHE_LOG_DIR}/web_delivery-www-access.log combined

                SSLEngine on
        SSLCertificateFile /etc/letsencrypt/live/some.site.com/fullchain.pem
                SSLCertificateKeyFile /etc/letsencrypt/live/some.site.com/privkey.pem
        </VirtualHost>
</IfModule>
```

Link

```
ln -s /etc/apache2/sites-available/www.mgrpropertyservices.com.conf /etc/apache2/sites-enabled
```

Run

```
# start
systemctl restart apache2.service

# open the fw
ufw allow 'Apache Secure'
```

## Redirector

### Install

```
apt install nginx -y

cd /etc/nginx

# rm default
rm sites-enabled/default
```

### Conf

```
# /etc/nginx/sites-available/attacker.conf
server {
    listen 443;
    server_name attacker.com;

    ssl on;
    ssl_certificate /etc/letsencrypt/live/attacker.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/attacker.com/privkey.pem;

    root /var/www/html;

    location /home {
        proxy_pass https://c2.attacker.com:7001 ;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    location /profile {
        proxy_pass https://c2.attacker.com:7001 ;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

Restart

```
# make available
ln -s /etc/nginx/sites-available/attacker.com.conf /etc/nginx/sites-enabled

systemctl restart nginx.service
```

## Covenant

```
# generate cert
certbot certonly --agree-tos --standalone -m robdahammer@gmail.com -d fw.vaultsec.xyz

# convert
cd /etc/letsencrypt/live/fw.vaultsec.xyz
openssl pkcs12 -export -out certificate.pfx -inkey privkey.pem -in cert.pem -certfile chain.pem

```