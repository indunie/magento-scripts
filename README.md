# Magento 2.4 Single Click Installation

This script will initilse new magento server with apache for ubuntu 18.04+. This script is intend to run on very new OS installation. If you use this in exsiting web server make sure to backup Apache configurations, MySql database etc.

This will install `ftp` server for magento home directory also.

## How to Install

Clone the git repository and go into cloned folder

```
git clone https://github.com/indunie/magento-install
cd magento-install

```


Then run the magento_install.sh script as root user with the following arguments. Change the argument values as necessary.

### Arguments

Every argument is optional and default value will be applied if ommited.

| Argument            | Description                                      | Default             |
|---------------------|--------------------------------------------------|---------------------|
| --magento-user      | Magento admin user                               | admin               |
| --magento-email     | Email for magento admin                          | admin@admin.com     |
| --magento-password  | Magento admin password                           | admin@123           |
| --database          | Magento database name                            | magentoip           |
| --database-user     | Magento database user                            | magentoip           |
| --database-password | Magento database password                        | magento@123         |
| --site-name         | Domain name or magento site name                 | mydomain.com        |
| --base-url          | Magento base URL                                 | http://mydomain.com |
| --system-user       | New system user for Magento <br>file permissions | magento             |
| --system-password   | Magento system user password                     | magento@123         |
| --elasticsearch-host| Elasticsearch host                               | localhost           |
| --elasticsearch-port| Elasticsearch port                               | 8080                |


system-user and system-password can be used to log into ftp server

### Example

 ```bash
  sudo ./magento_install.sh --magento-username admin \
    --magento-email admin@admin.com \
    --magento-password admin@123 \
    --database magento \
    --database-user magentoip \
    --database-password magento@123 \
    --site-name mydomain.com \
    --base-url http://mydomain.com \
    --system-user=magento \
    --system-password=magento@123
    --elasticsearch-host=localhost
    --elasticsearch-port=8080
```

It will take a few minutes to complete.

After installation, the admin url is printed as follows, Note the Magento Admin URI.

```
....
[SUCCESS]: Magento installation complete.
[SUCCESS]: Magento Admin URI: /admin_5da86s
Nothing to import.
```


### This will install/update following software
 
- Magento 2.4.2
- MySql 8.0.x
- PHP 7.4
- Elasticsearch 7.13.x
- Apache2
- Composer 2.x
 
## Creating replica/staging site from exsiting deployment

- Copy exsiting magento folder as new folder ( Make sure to clean any unnecesary/large log files inside `var/log`)
- Find current database used from `app/etc/env.php` in `db` section and replicate the database
   - To baclup databse run `mysqldump --single-transaction -u <database_user> -p <database_name> > <backup_db_name>-$(date +'%Y%m%d_%H%M%S').sql`
   - Then create a new database with desired name using mysql shell - Ex `create database my_staging_magento;`
- Restore previouse backup to new database.
   - Run following command
   ```bash
   mysql -u <db_user> -p <new_database_name> --init-command="SET SESSION FOREIGN_KEY_CHECKS=0;" < <backup_file_path.sql>
   ```
- Modify database name in new site by editing `app/etc/env.php`
- Update base URLs in new database
   - Execute mysql shell and select correct database
   - Find current base urls.
   
     `select * from core_config_data where path like '%base%url%';`
     
   - Update to new values. Make sure to have ending slash (`/`) and `https://`
   
     `update core_config_data set value = 'https://<new_domain>/' where value = 'https://<old_domain>/';`
     
   - Check again whether URLs are updated by 
      `select * from core_config_data where path like '%base%url%';`
      
- Point new DNS and create new apache configurations for new domain name(s) - for multisite magento deployments you need to create seperate config files for each new domain
   - Sample apache configuration file - let say new domain is `staging.example.com`. New config file is at `/etc/apache2/sites-available/staging.example.com.conf`. And make sure to replace necessary fields
   
   
   ```
   <VirtualHost *:80>
        # The ServerName directive sets the request scheme, hostname and port that
        # the server uses to identify itself. This is used when creating
        # redirection URLs. In the context of virtual hosts, the ServerName
        # specifies what hostname must appear in the request's Host: header to
        # match this virtual host. For the default virtual host (this file) this
        # value is not decisive as it is used as a last resort host regardless.
        # However, you must set it for any further virtual host explicitly.
        ServerName staging.example.com
        ServerAlias www.staging.example.com # Only add if you need www alias

        ServerAdmin webmaster@localhost
        DocumentRoot /home/magento/staging.example.com # Make sure to add correct path

        # Available loglevels: trace8, ..., trace1, debug, info, notice, warn,
        # error, crit, alert, emerg.
        # It is also possible to configure the loglevel for particular
        # modules, e.g.
        #LogLevel info ssl:warn

        ErrorLog ${APACHE_LOG_DIR}/staging.example.com.error.log
        CustomLog ${APACHE_LOG_DIR}/staging.example.com.access.log combined

        # For most configuration files from conf-available/, which are
        # enabled or disabled at a global level, it is possible to
        # include a line for only one particular virtual host. For example the
        # following line enables the CGI configuration for this host only
        # after it has been globally disabled with "a2disconf".
        #Include conf-available/serve-cgi-bin.conf

   </VirtualHost>

   <Directory "<path_to_magento_directory_withount_ending_slash>">
       AllowOverride All
   </Directory>
   ```
   
   - After modifying config files enable those configurations by running `a2ensite staging.example.com.conf` and run `systemctl reload apache2`
   - Edit `.htaccess` file for multi site deployments
   
- If all done correctly, new replica should work, check file permissions also
   
## Install SSL certificates

`certbot --apache -d <new-domain>`
## Install FileRun using docker and `docker-compose.yml`

```yml
version: '2'

services:
  db:
    image: mariadb:10.1
    environment:
      MYSQL_ROOT_PASSWORD: hsxsgW#5r
      MYSQL_USER: myuser
      MYSQL_PASSWORD: jhsbygF54Ff
      MYSQL_DATABASE: filerun
    volumes:
      - /filerun/db:/var/lib/mysql

  web:
    image: filerun/filerun
    environment:
      FR_DB_HOST: db
      FR_DB_PORT: 3306
      FR_DB_NAME: filerun
      FR_DB_USER: myuser
      FR_DB_PASS: jhsbygF54Ff
      APACHE_RUN_USER: magento
      APACHE_RUN_USER_ID: 1001
      APACHE_RUN_GROUP: magento
      APACHE_RUN_GROUP_ID: 1001
    depends_on:
      - db
    links:
      - db:db
    ports:
      - "8002:80"
    volumes:
      - ./html:/var/www/html
      - /home/magento:/user-files
```
