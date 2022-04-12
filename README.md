# Apache based ONTAP Configuration Backup

This repository has a high level example of how to configure Apache to accept POST based HTTPS ONTAP Configuration Backups.

**Advisory - This code is not production ready.  A number of things will need to be taken into account, for example using CA signed certificates, customize file names, security, etc...**

**This configuration is for demonstration purposes only**

The following configuration has been run after installing a fresh VM of Ubuntu 20.04

### Update the package sources list and update all the packages presently installed

```

sudo apt update && sudo apt upgrade -y

```

### Install Apache and Apache utilities

```

sudo apt install apache2 apache2-utils -y

```

### Install PHP module for Apache

```

sudo apt install libapache2-mod-php -y

```

### Disable default Apache site

```

sudo a2dissite 000-default

```

### Enable required Apache modules

```
sudo a2enmod ssl
sudo a2enmod dav
sudo a2enmod dav_fs
sudo a2enmod rewrite

```

### Create Backup Upload directory and set required permissions

```
sudo mkdir /var/www/netapp-config-backups
sudo chown www-data.www-data /var/www/netapp-config-backups

```

### Create DavLock directory and set required permissions

```
sudo mkdir /var/davlockdb
sudo chown www-data.www-data /var/davlockdb

```

### Create .htpasswd file

```

sudo htpasswd -c /etc/apache2/.htpasswd BackupUpload

```

Enter the password twice, as requested

### Create self signed certificates (You will probably want to use CA signed certificates instead)

```

sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/private/backup-upload.demo.netapp.com.key -out /etc/ssl/certs/backup-upload.demo.netapp.com.crt

```

Fill out the fields needed to create the certificate and private key

### Create the Apache configuration file

```

sudo tee -a /etc/apache2/sites-available/backup-upload.demo.netapp.com-ssl.conf >/dev/null <<'EOF'
<IfModule mod_ssl.c>
DavLockDB /var/davlockdb
  <VirtualHost _default_:443>
    ServerAdmin webmaster@demo.netapp.com
      Servername backup-upload.demo.netapp.com

      DocumentRoot /var/www/netapp-config-backups

      Alias /backups /var/www/netapp-config-backups

      <Location /backups>
        SSLRequireSSL
        Options Indexes FollowSymLinks MultiViews
        AuthType Basic
        AuthName BackupUpload
        AuthUserFile /etc/apache2/.htpasswd
        Require valid-user
        RewriteEngine On
        RewriteCond %{REQUEST_FILENAME} !-f
        RewriteCond %{REQUEST_FILENAME} !-d
        RewriteRule . /backups/index.php [L]
      </Location>


    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined

    SSLEngine on

    SSLCertificateFile      /etc/ssl/certs/backup-upload.demo.netapp.com.crt
    SSLCertificateKeyFile   /etc/ssl/private/backup-upload.demo.netapp.com.key

  </VirtualHost>
</IfModule>

# vim: syntax=apache ts=4 sw=4 sts=4 sr noet
EOF

```

### Create PHP index.php file

```

sudo tee -a /var/www/netapp-config-backups/index.php >/dev/null <<'EOF'
<?php
$file_upload_directory = "/var/www/netapp-config-backups";
$target_path = $file_upload_directory ."/". basename( $_SERVER['REQUEST_URI']);

$file_input_stream = file_get_contents('php://input', false, null, 0, $_SERVER['CONTENT_LENGTH']);
if(file_put_contents($target_path, $file_input_stream)) {
        echo "The file ". basename($_SERVER['REQUEST_URI']) ." has been uploaded";
} else {
        echo "There was an error uploading the file, please try again!";
}
?>
EOF

```

### Enable Apache configuration

```

sudo a2ensite backup-upload.demo.netapp.com-ssl.conf

```

### Restart Apache services

```

sudo systemctl restart apache2

```

### Upload Configuration Backups from ONTAP

**As we are using a self signed certificate we need to force -validate-certification to false**

<p align="center">
<img src="https://github.com/MrStevenSmith/Apache-based-ONTAP-Config-Backups/blob/main/images/ONTAP-Config-Backup.png">
</p>


<p align="center">
<img src="https://github.com/MrStevenSmith/Apache-based-ONTAP-Config-Backups/blob/main/images/Linux-Config-Backup.png">
</p>

## License

This project is licensed under the GNU GENERAL PUBLIC LICENSE - see the [LICENSE](https://github.com/MrStevenSmith/Trident-WordPress-Application/blob/master/LICENSE) file for details
<br />
<br />
## Author

* **Steven Smith** - [MrStevenSmith](https://github.com/MrStevenSmith)