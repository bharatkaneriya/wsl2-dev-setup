# WSL2 Dev Environment Setup Guide
> Ubuntu 24.04 LTS + Apache + PHP-FPM (8.1 & 8.4) + MySQL + PostgreSQL + Redis + Node

---

## 1. Windows — WSL2 Installation

Open **PowerShell as Administrator**:

```powershell
wsl --install -d Ubuntu-24.04
wsl --set-default-version 2
```

Restart when prompted. After restart Ubuntu opens — set username and password.

---

## 2. Windows — RAM Limit (important for 8GB machines)

```powershell
notepad $env:USERPROFILE\.wslconfig
```

Paste:

```ini
[wsl2]
memory=3GB
processors=2
swap=2GB

[experimental]
autoMemoryReclaim=gradual
```

Save and restart WSL2:

```powershell
wsl --shutdown
```

Reopen Ubuntu terminal.

---

## 3. Update Ubuntu & Install Essentials

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget git unzip zip build-essential software-properties-common
```

---

## 4. Add PHP Repository (Ondrej PPA)

```bash
sudo add-apt-repository ppa:ondrej/php -y
sudo apt update
```

---

## 5. Install PHP 8.1 & 8.4 with FPM

```bash
# PHP 8.1 — for old CI3 and legacy projects
sudo apt install -y php8.1 php8.1-fpm php8.1-cli \
  php8.1-mysql php8.1-pgsql php8.1-curl php8.1-mbstring \
  php8.1-xml php8.1-zip php8.1-bcmath php8.1-gd \
  php8.1-intl php8.1-redis php8.1-opcache

# PHP 8.4 — for Laravel, WordPress, new projects
sudo apt install -y php8.4 php8.4-fpm php8.4-cli \
  php8.4-mysql php8.4-pgsql php8.4-curl php8.4-mbstring \
  php8.4-xml php8.4-zip php8.4-bcmath php8.4-gd \
  php8.4-intl php8.4-redis php8.4-opcache
```

Verify:

```bash
php8.1 -v && php8.4 -v
```

---

## 6. Install Apache

```bash
sudo apt install -y apache2

# Enable required modules
sudo a2enmod proxy_fcgi setenvif rewrite headers

# Enable PHP-FPM configs
sudo a2enconf php8.1-fpm
sudo a2enconf php8.4-fpm

# Start Apache
sudo service apache2 start
```

### Default vhost — serves /var/www/html on port 80

```bash
sudo nano /etc/apache2/sites-available/000-default.conf
```

```apache
<VirtualHost *:80>
    ServerName localhost
    DocumentRoot /var/www/html

    <Directory /var/www/html>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

```bash
sudo service apache2 restart
```

---

## 7. Install MySQL

```bash
sudo apt install -y mysql-server
sudo service mysql start
sudo mysql_secure_installation
```

Answer prompts:

| Question | Answer |
|---|---|
| Validate password? | N |
| Remove anonymous users? | Y |
| Disallow remote root? | Y |
| Remove test database? | Y |
| Reload privileges? | Y |

Set root password and create dev user:

```bash
sudo mysql
```

```sql
ALTER USER 'root'@'localhost' IDENTIFIED WITH caching_sha2_password BY 'root';
CREATE USER 'devuser'@'localhost' IDENTIFIED WITH caching_sha2_password BY 'devpass';
GRANT ALL PRIVILEGES ON *.* TO 'devuser'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

---

## 8. Install PostgreSQL

```bash
sudo apt install -y postgresql postgresql-contrib
sudo service postgresql start
sudo -u postgres psql
```

```sql
CREATE USER devuser WITH PASSWORD 'devpass';
ALTER USER devuser CREATEDB;
\q
```

---

## 9. Install Redis

```bash
sudo apt install -y redis-server
sudo service redis-server start
redis-cli ping
```

Should reply `PONG`.

---

## 10. Install Composer

```bash
curl -sS https://getcomposer.org/installer | php
sudo mv composer.phar /usr/local/bin/composer
composer --version
```

---

## 11. Install Node via nvm

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
source ~/.bashrc
nvm install 20
nvm use 20
nvm alias default 20
node -v && npm -v
```

---

## 12. Install phpMyAdmin

```bash
sudo apt install -y phpmyadmin
```

Prompts:
- Web server → select `apache2` with spacebar → Tab → OK
- Configure with dbconfig → Yes
- MySQL password → `root`

Link to web root:

```bash
sudo ln -s /usr/share/phpmyadmin /var/www/html/phpmyadmin
```

Visit: `http://localhost/phpmyadmin`

---

## 13. Install Adminer

```bash
wget -O /var/www/html/adminer.php https://www.adminer.org/latest.php
sudo chown www-data:www-data /var/www/html/adminer.php
```

Visit: `http://localhost/adminer.php`

---

## 14. Project Directory Permissions

### Set up /var/www/html ownership

```bash
sudo chown -R $USER:www-data /var/www/html
sudo chmod -R 755 /var/www/html
sudo chmod g+s /var/www/html
```

### For any new project folder

```bash
# Replace 'myproject' with your project folder name
sudo chown -R $USER:www-data /var/www/html/myproject
sudo find /var/www/html/myproject -type d -exec chmod 755 {} \;
sudo find /var/www/html/myproject -type f -exec chmod 644 {} \;
```

### Laravel specific

```bash
sudo chmod -R 775 /var/www/html/myproject/storage
sudo chmod -R 775 /var/www/html/myproject/bootstrap/cache
chmod 600 /var/www/html/myproject/.env
```

### WordPress specific

```bash
sudo chmod -R 775 /var/www/html/myproject/wp-content/uploads
chmod 600 /var/www/html/myproject/wp-config.php
```

---

## 15. Git Setup

```bash
git config --global user.name "Your Name"
git config --global user.email "your@email.com"
git config --global init.defaultBranch main
git config --global pull.rebase true
```

### SSH key (recommended — no password prompts)

```bash
ssh-keygen -t ed25519 -C "your@email.com"
cat ~/.ssh/id_ed25519.pub
```

Add the output to: https://github.com/settings/ssh/new

Test:

```bash
ssh -T git@github.com
```

---

## 16. Startup Script

```bash
nano ~/start-dev.sh
```

```bash
#!/bin/bash
sudo service mysql start
sudo service postgresql start
sudo service redis-server start
sudo service php8.1-fpm start
sudo service php8.4-fpm start
sudo service apache2 start
echo "✓ All dev services started"
```

```bash
chmod +x ~/start-dev.sh
```

Run every time you open WSL2:

```bash
~/start-dev.sh
```

Shutdown when done for the day (frees RAM):

```powershell
wsl --shutdown
```

---

## 17. New Project Vhost Template

For each project that needs its own domain, create a vhost file:

```bash
sudo nano /etc/apache2/sites-available/PROJECTNAME.test.conf
```

```apache
<VirtualHost *:80>
    ServerName PROJECTNAME.test
    DocumentRoot /var/www/html/PROJECTFOLDER

    # PHP 8.1 for old/CI3 projects:
    # SetHandler "proxy:unix:/run/php/php8.1-fpm.sock|fcgi://localhost"

    # PHP 8.4 for Laravel/WordPress/new projects:
    <FilesMatch \.php$>
        SetHandler "proxy:unix:/run/php/php8.4-fpm.sock|fcgi://localhost"
    </FilesMatch>

    <Directory /var/www/html/PROJECTFOLDER>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/PROJECTNAME-error.log
    CustomLog ${APACHE_LOG_DIR}/PROJECTNAME-access.log combined
</VirtualHost>
```

Enable it:

```bash
sudo a2ensite PROJECTNAME.test.conf
sudo service apache2 reload
```

Add to Windows hosts file (`C:\Windows\System32\drivers\etc\hosts`):

```
127.0.0.1  PROJECTNAME.test
```

---

## 18. PHP Version Reference

| Project type | PHP version | Socket |
|---|---|---|
| CI3 / legacy projects | 8.1 | `php8.1-fpm.sock` |
| Laravel / WordPress | 8.4 | `php8.4-fpm.sock` |
| New projects | 8.4 | `php8.4-fpm.sock` |

Switch PHP version in vhost by changing one line:

```apache
# PHP 8.1
SetHandler "proxy:unix:/run/php/php8.1-fpm.sock|fcgi://localhost"

# PHP 8.4
SetHandler "proxy:unix:/run/php/php8.4-fpm.sock|fcgi://localhost"
```

---

## 19. Quick Reference

| Tool | URL / Command |
|---|---|
| phpMyAdmin | http://localhost/phpmyadmin |
| Adminer | http://localhost/adminer.php |
| MySQL user | devuser / devpass |
| PostgreSQL user | devuser / devpass |
| Projects folder | /var/www/html/ |
| Apache vhosts | /etc/apache2/sites-available/ |
| PHP 8.1 config | /etc/php/8.1/fpm/php.ini |
| PHP 8.4 config | /etc/php/8.4/fpm/php.ini |
| Apache error log | /var/log/apache2/error.log |
| Start all services | ~/start-dev.sh |
| Stop WSL2 | wsl --shutdown (PowerShell) |

---

## 20. Import Database

MySQL:

```bash
# Create database first
mysql -u devuser -p -e "CREATE DATABASE dbname;"

# Import from Windows path
mysql -u devuser -p dbname < /mnt/c/Users/YourName/Downloads/file.sql
```

PostgreSQL:

```bash
createdb -U devuser -h 127.0.0.1 dbname
psql -U devuser -h 127.0.0.1 dbname < /mnt/c/Users/YourName/Downloads/file.sql
```

