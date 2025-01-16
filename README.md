# Daloradius Installation on Ubuntu Server

### Updating System
```bash
sudo apt update && sudo apt -y upgrade
[ -f /var/run/reboot-required ] && sudo reboot -f
```

### Install Apache and PHP
```bash
sudo apt -y install apache2
sudo apt -y install nano php libapache2-mod-php php-{gd,common,mail,mail-mime,mysql,pear,db,mbstring,xml,curl,zip}
php -v
```

### Install MariaDB and Create Database
```bash
sudo apt update && sudo apt install mariadb-server -y
sudo mysql_secure_installation
# Set a strong root password
sudo mysql -u root -p
```

#### Create Database
```sql
CREATE DATABASE radius;
GRANT ALL ON radius.* TO YOUR_USERNAME@localhost IDENTIFIED BY "STRONG_PASSWORD";
FLUSH PRIVILEGES;
QUIT
```

### Install and Configure FreeRADIUS
```bash
sudo apt policy freeradius
sudo apt -y install freeradius freeradius-mysql freeradius-utils
sudo su -
mysql -u root -p radius < /etc/freeradius/3.0/mods-config/sql/main/mysql/schema.sql
sudo mysql -u root -p -e "use radius;show tables;"
sudo ln -s /etc/freeradius/3.0/mods-available/sql /etc/freeradius/3.0/mods-enabled/
```

#### Edit SQL Configuration
```bash
sudo nano /etc/freeradius/3.0/mods-enabled/sql
```
Update the file with:
```text
sql {
    driver = "rlm_sql_mysql"
    dialect = "mysql"

    server = "localhost"
    port = 3306
    login = "YOUR_USERNAME"
    password = "STRONG_PASSWORD"

    radius_db = "radius"

    read_clients = yes
    client_table = "nas"
}
```

#### Comment Out TLS Section
```text
mysql {
    # Comment out TLS section:
    # tls {
    #   ca_file = "/etc/ssl/certs/my_ca.crt"
    #   ...
    # }
    warnings = auto
}
```

#### Set Permissions and Restart FreeRADIUS
```bash
sudo chgrp -h freerad /etc/freeradius/3.0/mods-available/sql
sudo chown -R freerad:freerad /etc/freeradius/3.0/mods-enabled/sql
sudo systemctl restart freeradius.service
```

### Install and Configure Daloradius
```bash
sudo apt -y install git
git clone https://github.com/lirantal/daloradius.git
sudo su -
mysql -u root -p radius < daloradius/contrib/db/fr3-mariadb-freeradius.sql
mysql -u root -p radius < daloradius/contrib/db/mariadb-daloradius.sql
sudo mv daloradius /var/www/
```

#### Configure Daloradius
```bash
cd /var/www/daloradius/app/common/includes/
sudo cp daloradius.conf.php.sample daloradius.conf.php
sudo chown www-data:www-data daloradius.conf.php
sudo nano daloradius.conf.php
```
Update the file with:
```php
$configValues['CONFIG_DB_HOST'] = 'localhost';
$configValues['CONFIG_DB_PORT'] = '3306';
$configValues['CONFIG_DB_USER'] = 'YOUR_USERNAME';
$configValues['CONFIG_DB_PASS'] = 'STRONG_PASSWORD';
$configValues['CONFIG_DB_NAME'] = 'radius';
```
Create necessary directories:
```bash
cd /var/www/daloradius/
sudo mkdir -p var/{log,backup}
sudo chown -R www-data:www-data var
```

### Configure Apache Web Server
```bash
sudo tee /etc/apache2/ports.conf<<EOF
Listen 80
Listen 8000

<IfModule ssl_module>
    Listen 443
</IfModule>

<IfModule mod_gnutls.c>
    Listen 443
</IfModule>
EOF

sudo tee /etc/apache2/sites-available/operators.conf<<EOF
<VirtualHost *:8000>
    ServerAdmin operators@localhost
    DocumentRoot /var/www/daloradius/app/operators

    <Directory /var/www/daloradius/app/operators>
        Options -Indexes +FollowSymLinks
        AllowOverride None
        Require all granted
    </Directory>

    <Directory /var/www/daloradius>
        Require all denied
    </Directory>

    ErrorLog \${APACHE_LOG_DIR}/daloradius/operators/error.log
    CustomLog \${APACHE_LOG_DIR}/daloradius/operators/access.log combined
</VirtualHost>
EOF

sudo tee /etc/apache2/sites-available/users.conf<<EOF
<VirtualHost *:80>
    ServerAdmin users@localhost
    DocumentRoot /var/www/daloradius/app/users

    <Directory /var/www/daloradius/app/users>
        Options -Indexes +FollowSymLinks
        AllowOverride None
        Require all granted
    </Directory>

    <Directory /var/www/daloradius>
        Require all denied
    </Directory>

    ErrorLog \${APACHE_LOG_DIR}/daloradius/users/error.log
    CustomLog \${APACHE_LOG_DIR}/daloradius/users/access.log combined
</VirtualHost>
EOF

sudo a2ensite users.conf operators.conf
sudo mkdir -p /var/log/apache2/daloradius/{operators,users}
sudo a2dissite 000-default.conf
sudo systemctl restart apache2 freeradius
```

### Adding HTTPS Support (Self-Signed Certificate)
#### Generate a Self-Signed Certificate
```bash
sudo mkdir /etc/ssl/daloradius
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/daloradius/daloradius.key -out /etc/ssl/daloradius/daloradius.crt
```
Fill in the requested details. Use the server's IP address for the **Common Name (CN)** if you don’t have a domain.

#### Configure Apache for HTTPS
```bash
sudo nano /etc/apache2/sites-available/daloradius-ssl.conf
```
Add the following configuration:
```apache
<VirtualHost *:443>
    ServerAdmin admin@localhost
    DocumentRoot /var/www/daloradius

    SSLEngine on
    SSLCertificateFile /etc/ssl/daloradius/daloradius.crt
    SSLCertificateKeyFile /etc/ssl/daloradius/daloradius.key

    <Directory /var/www/daloradius>
        Options -Indexes +FollowSymLinks
        AllowOverride None
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/daloradius_ssl_error.log
    CustomLog ${APACHE_LOG_DIR}/daloradius_ssl_access.log combined
</VirtualHost>
```

#### Enable HTTPS Configuration
```bash
sudo a2enmod ssl
sudo a2ensite daloradius-ssl.conf
sudo systemctl restart apache2
```

#### Redirect HTTP to HTTPS
Modify the HTTP configuration file to redirect traffic:
```bash
sudo nano /etc/apache2/sites-available/users.conf
```
Add:
```apache
<VirtualHost *:80>
    Redirect permanent / https://<YOUR_SERVER_IP>/
</VirtualHost>
```
Restart Apache:
```bash
sudo systemctl restart apache2
```

### Login
- Access the web interface at: `https://<YOUR_SERVER_IP>:443/`
- Default Daloradius credentials:
  - **Username**: administrator
  - **Password**: radius

### Troubleshooting
If any of the services couldn’t be restarted:
- Reboot the system and try:
  ```bash
  sudo systemctl status <apache2|mysql|freeradius>
  sudo systemctl start <apache2|mysql|freeradius>
  ```

### Backing Up Daloradius
#### Backup the Database
```bash
mysqldump -u root -p radius > ~/radius_backup.sql
```

#### Backup Daloradius Files
```bash
sudo tar -czvf ~/daloradius_files_backup.tar.gz /var/www/daloradius
```

#### Backup Apache Configuration
```bash
sudo tar -czvf ~/apache_configs_backup.tar.gz /etc/apache2/sites-available /etc/apache2/sites-enabled
```
