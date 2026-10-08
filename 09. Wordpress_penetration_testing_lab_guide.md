# Building an Authorized WordPress Training Lab

> **Educational and authorized use only.** Build this environment on a local virtual machine, an isolated host-only network, or a disposable container project. Do not expose intentionally vulnerable software to the public Internet. Use strong, unique credentials for any non-laboratory system.

## Learning goals

This guide explains three controlled ways to create a WordPress practice environment: a traditional Linux installation using Apache, MariaDB, and PHP; a containerized installation using Docker Compose; and a Windows installation using a local web stack. The commands are accompanied by representative output and an explanation of what each result means.

The examples use a database named `wordpress`, a database user named `wp_user`, and local access only. These values are suitable for a disposable lab but should not be reused in production. Screenshots are intentionally represented as text descriptions or ASCII diagrams so that the guide remains portable and does not depend on external image links.

## Architecture overview

```text
+----------------------+       HTTP        +------------------------+
| Browser              | ----------------> | Apache / WordPress     |
| localhost            |                   | PHP application        |
+----------------------+                   +-----------+------------+
                                                       |
                                                       | SQL
                                                       v
                                           +------------------------+
                                           | MariaDB / MySQL        |
                                           | wordpress database     |
                                           +------------------------+
```

The browser requests WordPress pages from the web server. PHP executes the application, while MariaDB or MySQL stores posts, accounts, settings, and metadata. Separating these roles helps explain where configuration problems occur: HTTP errors usually involve the web server or PHP, while database connection errors involve credentials, hostnames, ports, or permissions.

## Safety and prerequisites

Use a disposable virtual machine or a private training network. Take a snapshot before changing the system. Do not install intentionally vulnerable plugins on a production server, and do not forward the lab's web port from a router or public cloud firewall.

Check the operating system and current identity:

```bash
cat /etc/os-release
id
```

Representative output:

```text
PRETTY_NAME="Ubuntu 22.04.5 LTS"
uid=1000(student) gid=1000(student) groups=1000(student),27(sudo)
```

The first command identifies the distribution and version. The `id` command shows the current user, numeric IDs, and groups. Membership in `sudo` means the user may request administrative privileges, subject to the system's policy.

## Method A: Linux package installation

### Install Apache

```bash
sudo apt update
sudo apt install apache2
```

Representative output:

```text
Reading package lists... Done
Building dependency tree... Done
The following NEW packages will be installed:
  apache2 apache2-bin apache2-data
Setting up apache2 ...
```

`apt update` refreshes the local package index. `apt install apache2` installs the web server and supporting packages. The exact package list varies by distribution. After installation, verify the service:

```bash
sudo systemctl enable --now apache2
sudo systemctl status apache2 --no-pager
```

Example output:

```text
● apache2.service - The Apache HTTP Server
     Loaded: loaded (...; enabled)
     Active: active (running)
```

`enable --now` configures Apache to start at boot and starts it immediately. `Active: active (running)` indicates that the service is currently running. If the service fails, inspect logs rather than repeatedly restarting it.

Test the local HTTP endpoint:

```bash
curl -I http://127.0.0.1/
```

Representative output:

```text
HTTP/1.1 200 OK
Server: Apache/2.4.x
Content-Type: text/html; charset=UTF-8
```

The `200 OK` status confirms that Apache returned a response. The server header provides software information but should not be treated as a complete security assessment.

### Install MariaDB

```bash
sudo apt install mariadb-server mariadb-client
sudo systemctl enable --now mariadb
```

Representative output:

```text
The following NEW packages will be installed:
  mariadb-client mariadb-server
Created symlink ... mariadb.service ...
```

The server package provides the database daemon, while the client package provides command-line tools such as `mariadb` or `mysql`. Enabling and starting the service makes the database available locally.

Run the hardening assistant:

```bash
sudo mariadb-secure-installation
```

Depending on the version, the command may be named `mysql_secure_installation`. A typical sequence is:

```text
Enter current password for root (enter for none):  [press Enter]
Set root password? [Y/n]: Y
Remove anonymous users? [Y/n]: Y
Disallow root login remotely? [Y/n]: Y
Remove test database and access to it? [Y/n]: Y
Reload privilege tables now? [Y/n]: Y
```

The prompts remove anonymous accounts, prevent remote database-root login, remove the test database, and reload privilege information. Choose a strong administrator password when the installation asks for one. On newer MariaDB systems, root may use socket authentication, so the exact prompts and login behavior can differ.

### Install PHP

```bash
sudo apt install php php-mysql libapache2-mod-php
sudo systemctl restart apache2
```

Representative output:

```text
Setting up php ...
Setting up libapache2-mod-php ...
Processing triggers for libapache2-mod-php ...
```

The PHP package supplies the interpreter, `php-mysql` provides database connectivity, and the Apache module allows Apache to pass PHP files to the interpreter. Restarting Apache loads the new module configuration.

Verify the installed components:

```bash
php -v
php -m | grep -E 'mysqli|PDO|mysql'
```

Example output:

```text
PHP 8.1.x (cli) ...
mysqli
PDO
pdo_mysql
```

`php -v` shows the interpreter version. The module query confirms that PHP can communicate with MySQL-compatible databases. Version output is informative; keep the lab isolated if the software is intentionally old or vulnerable.

## Create a WordPress database

Open the database client as an administrator:

```bash
sudo mariadb
```

Run statements that create a dedicated database and least-privilege application user:

```sql
CREATE DATABASE wordpress CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'wp_user'@'localhost' IDENTIFIED BY 'Use-a-unique-lab-password';
GRANT ALL PRIVILEGES ON wordpress.* TO 'wp_user'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

Representative output:

```text
Query OK, 1 row affected
Query OK, 0 rows affected
Query OK, 0 rows affected
Query OK, 0 rows affected
Bye
```

The database stores WordPress data. The application account is restricted to the local host and receives privileges only on the WordPress database, rather than global database-administrator privileges. `FLUSH PRIVILEGES` makes privilege-table changes available immediately; modern `CREATE USER` and `GRANT` statements generally update privileges without requiring it, but it is harmless in this lab sequence.

Verify the account without exposing its password:

```bash
sudo mariadb -e "SHOW DATABASES; SELECT User,Host FROM mysql.user WHERE User='wp_user';"
```

Example output:

```text
Database
information_schema
mysql
performance_schema
wordpress

User    Host
wp_user localhost
```

The output confirms that the database exists and that the application account is tied to `localhost`. The password is not printed, which is safer for terminal history and screenshots.

## Download and install WordPress

Use HTTPS and verify the downloaded archive in a controlled lab:

```bash
cd /tmp
curl -LO https://wordpress.org/latest.tar.gz
tar -xzf latest.tar.gz
sudo mv wordpress /var/www/html/wordpress
```

Representative output:

```text
% Total    % Received    % Xferd  Average Speed
100  ...   100  ...      ...
```

`curl -LO` downloads the archive while preserving the remote filename. `tar -xzf` extracts a gzip-compressed tar archive. Moving the directory into `/var/www/html` places it under Apache's usual document root. In a production workflow, verify the release signature or checksum according to the vendor's documentation before deployment.

Set ownership and conservative permissions:

```bash
sudo chown -R www-data:www-data /var/www/html/wordpress
sudo find /var/www/html/wordpress -type d -exec chmod 755 {} \\\;
sudo find /var/www/html/wordpress -type f -exec chmod 644 {} \\\;
sudo mkdir -p /var/www/html/wordpress/wp-content/uploads
sudo chown -R www-data:www-data /var/www/html/wordpress/wp-content/uploads
```

Representative output:

```text
# No output indicates that the commands completed successfully.
```

`www-data:www-data` makes Apache's service account the owner. Directory mode `755` permits traversal and reading while limiting modifications to the owner; file mode `644` permits reading but limits writing. Uploads require special care because web-writable directories can become a security boundary. Do not make the entire WordPress tree world-writable.

## Complete browser installation

Open the local installer:

```text
http://127.0.0.1/wordpress/
```

The installer normally asks for:

- Database name: `wordpress`.
- Database user: `wp_user`.
- Database password: the unique lab password created earlier.
- Database host: `localhost`.
- Table prefix: a lab-specific value such as `wp_`.

The database credentials connect WordPress to MariaDB. They are different from the WordPress administrator credentials created later in the browser. A successful connection leads to the site information page, where you choose the site title and create an application administrator.

A useful verification command after installation is:

```bash
curl -I http://127.0.0.1/wordpress/
```

Example output:

```text
HTTP/1.1 200 OK
Content-Type: text/html; charset=UTF-8
```

A `200` response means Apache and WordPress returned a page. A `500` response often suggests a PHP error, missing module, permissions problem, or configuration issue. A database connection error usually indicates incorrect database name, user, password, host, or service status.

## Configure the site URL carefully

WordPress stores site URLs in its configuration and database. In a local lab, you can define them explicitly in `wp-config.php`:

```php
define('WP_HOME', 'http://127.0.0.1/wordpress');
define('WP_SITEURL', 'http://127.0.0.1/wordpress');
```

These values tell WordPress where the site is hosted and where its core files are located. Hard-coding a loopback address makes the site available only from the local machine. If you use a private lab hostname, use that hostname consistently instead. Do not construct site URLs from an untrusted HTTP Host header on a public deployment.

## Method B: Docker Compose

Containerization packages the application and database into separate, reproducible services. Create a disposable project directory:

```bash
sudo apt update
sudo apt install docker.io docker-compose-plugin
sudo systemctl enable --now docker
mkdir -p ~/wordpress-lab
cd ~/wordpress-lab
```

Verify Docker:

```bash
docker --version
docker compose version
```

Example output:

```text
Docker version 26.x, build ...
Docker Compose version v2.x
```

The version commands confirm that the engine and Compose plugin are available. Package names differ across distributions; use the package format supported by your operating system.

Create `compose.yaml`:

```yaml
services:
  db:
    image: mariadb:10.11
    restart: unless-stopped
    environment:
      MARIADB_DATABASE: wordpress
      MARIADB_USER: wordpress
      MARIADB_PASSWORD: change-this-lab-password
      MARIADB_ROOT_PASSWORD: change-this-root-password
    volumes:
      - db_data:/var/lib/mysql

  wordpress:
    image: wordpress:latest
    depends_on:
      - db
    ports:
      - "127.0.0.1:8000:80"
    restart: unless-stopped
    environment:
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_USER: wordpress
      WORDPRESS_DB_PASSWORD: change-this-lab-password
      WORDPRESS_DB_NAME: wordpress

volumes:
  db_data:
```

The database service stores its data in a named volume. The WordPress service connects to the database through the internal service name `db` on port 3306. The port mapping binds host loopback port 8000 to container port 80, preventing direct exposure on external interfaces. Replace the example passwords with unique lab-only values.

Start the stack:

```bash
docker compose up -d
```

Representative output:

```text
[+] Running 3/3
 ✔ Network wordpress-lab_default  Created
 ✔ Container wordpress-lab-db-1  Started
 ✔ Container wordpress-lab-wordpress-1  Started
```

The `-d` option runs containers in detached mode. The network allows the services to communicate, and the named volume preserves database data when containers are recreated.

Inspect status and logs:

```bash
docker compose ps
docker compose logs --tail=30 wordpress
```

Example output:

```text
NAME                         SERVICE     STATUS          PORTS
wordpress-lab-db-1           db          running
wordpress-lab-wordpress-1    wordpress   running         127.0.0.1:8000->80/tcp
```

The status output confirms whether each service is running and shows the host-to-container port mapping. Logs help identify startup failures, PHP errors, missing database connectivity, or permission issues. Never publish database passwords from logs or screenshots.

Open the browser installer at:

```text
http://127.0.0.1:8000/
```

The database host must be `db:3306`, not `127.0.0.1`, because WordPress runs in a separate container. Inside the WordPress container, `127.0.0.1` refers to the WordPress container itself, while `db` resolves to the database service through the Compose network.

Stop and remove the lab stack when finished:

```bash
docker compose down
```

Typical output:

```text
[+] Running 2/2
 ✔ Container wordpress-lab-wordpress-1  Removed
 ✔ Container wordpress-lab-db-1         Removed
```

This removes containers and the network but normally keeps the named volume. To remove the database volume as well, use `docker compose down -v` only when you intentionally want to destroy the lab data.

## Method C: Windows with a local stack

A Windows local stack can provide Apache, PHP, and MariaDB/MySQL through a desktop control panel. Keep the control panel bound to localhost or a private training network. Place the extracted WordPress directory under the stack's document root, commonly a path such as `C:\xampp\htdocs\wordpress`.

Confirm the services in the control panel or with a browser request:

```text
http://127.0.0.1/
```

The control panel should show Apache and the database service as running. A browser response from the local address confirms that the web server is reachable, but it does not prove that the database or PHP integration is configured correctly.

Open the database administration interface only locally, for example:

```text
http://127.0.0.1/phpmyadmin/
```

Create a database named `wordpress`, then open:

```text
http://127.0.0.1/wordpress/
```

Use the database name, username, password, and host configured by your local stack. Some default installations use a local administrative account with an empty password; this is acceptable only for a disposable offline lab and should never be copied to production.

## Optional vulnerable components

A penetration-testing lab may deliberately install outdated or vulnerable plugins, but this should be done only on an isolated snapshot. Do not use real credentials, real content, public DNS, port forwarding, or shared networks. Record the plugin versions and remove the components when the exercise ends.

A safer training approach is to use intentionally vulnerable practice applications or a dedicated lab image rather than weakening a normal website. Keep the host firewall enabled and bind local services to loopback whenever remote access is unnecessary.

## Troubleshooting guide

### Apache returns 403

Check ownership and directory permissions:

```bash
namei -l /var/www/html/wordpress
sudo apachectl configtest
```

Example output:

```text
Syntax OK
```

`namei -l` shows permissions for every directory component in the path. `apachectl configtest` validates Apache configuration syntax. A `Syntax OK` result means syntax is valid, not that every application permission or runtime dependency is correct.

### PHP code downloads instead of executing

Check that PHP and the Apache PHP module are installed and reload Apache:

```bash
php -v
sudo systemctl restart apache2
sudo journalctl -u apache2 --since "10 minutes ago" --no-pager
```

The log output may identify a missing module, syntax error, or configuration issue. Do not display production secrets while collecting diagnostic evidence.

### WordPress cannot connect to the database

Check service status and test the database account:

```bash
sudo systemctl status mariadb --no-pager
mariadb -u wp_user -p -h 127.0.0.1 wordpress -e 'SELECT 1;'
```

Expected output:

```text
1
1
```

The first command checks whether MariaDB is running. The second authenticates to the named database and executes a harmless query. Failure may indicate an incorrect password, host restriction, missing database, inactive service, or a port mismatch.

For Docker Compose, use the service name and execute the test from the application environment:

```bash
docker compose exec wordpress getent hosts db
docker compose ps
```

Example output:

```text
172.20.0.2      db
NAME                         SERVICE     STATUS
wordpress-lab-db-1           db          running
wordpress-lab-wordpress-1    wordpress   running
```

The `getent` result proves that the Compose network resolves `db`. The status output confirms that both containers are running.

## Shutdown and cleanup

For a package-based lab, disable services when not in use:

```bash
sudo systemctl disable --now apache2 mariadb
```

Representative output:

```text
Removed ... apache2.service
Stopped apache2.service
Stopped mariadb.service
```

This stops the web and database services and prevents automatic startup. Remove the lab snapshot or virtual machine when the exercise is complete. For Docker, use `docker compose down`, and remove the volume only if the stored database data is no longer needed.

## Key lessons

A WordPress lab has three logical layers: the web server, the PHP application, and the database. Successful setup requires matching database credentials, correct service status, valid filesystem permissions, and a reachable local URL. Containerized deployment reduces manual dependency work and improves repeatability, while a package-based installation makes each layer visible for learning and troubleshooting.
