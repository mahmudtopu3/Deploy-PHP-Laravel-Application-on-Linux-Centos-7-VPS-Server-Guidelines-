

# Deploy-PHP-Laravel-Application-on-Linux-Centos-7-VPS-Server-Guidelines-
Create **Digital ocean** instance on https://m.do.co/c/1c72f7d5845d and get **100 USD Free Credit** !
# Initial Setup

> Here we will setup swap file and other necessary things before
> deploying laravel application

### Create Swap File
Check whether you have swap file or not. Run following command. 
	

    free -m
 Or
   

     swapon -s 
If you see nothing or swap data empty, then create a swap file first. Run following commands.

    dd if=/dev/zero of=/swapfile count=1024 bs=1M
Here we may create a swapfile having double size of the physical ram. So here in the above command count=1024 is used for 1GB size. If you need 2GB then count will be 2048 and so on. 
Now  setup the swap file we just created 

    mkswap /swapfile
Enable swapfile 

    swapon /swapfile
Now check the swap size again

    free -m
To enable swapfile after rebooting every time we need to edit the file system table file 

    nano /etc/fstab
Add the following line at the end of the file
> /swapfile   none    swap    sw    0   0

Now save by pressing ctrl + x, then press y and enter. 
If you see any error opening with nano can use vim or install **nano** if not available by following command 

    sudo yum install nano

## Install and configure Apache Server, PHP and its extensions, phpmyAdmin 
Now on centos 7 update repository to get the latest packages by running following commands. 

    rpm -Uvh https://download-ib01.fedoraproject.org/pub/epel/7/x86_64/Packages/e/epel-release-7-13.noarch.rpm
    
    rpm -Uvh http://rpms.famillecollet.com/enterprise/remi-release-7.rpm
    
    sudo yum update 

Here after updating, we will install apache from remi,epel repository 
### Install apache2 server
    yum --enablerepo=remi,epel install httpd
### Install  mysql server [mariadb]

    yum --enablerepo=remi,epel install mysql-server 
It may give you error as from centos 7 all mysql repositories are replaced with mariadb. 

    yum --enablerepo=remi,epel install mariadb-server
Now to run mariadb, run following command 

    service mariadb start 
To check if  mysql works, you can type **mysql** in the terminal and can login automatically and run sql commands also. But this is not a secured way , so we will secure mysql by giving password and set other options. You will be asked whether you need remote location, remove test users, test databases etc. It depends on you. We will allow remote connection here. To secure run this command. Enter blank password and set a password for root user.

    /usr/bin/mysql_secure_installation
Now you can login to mysql terminal by running this command and give password to login securely. 

    mysql -uroot -p 

### Install PHP 7.4 and some required extensions on centos 7 
Before installing php, we will install yum utils which can handle additional package's extensions. 

    sudo yum -y install yum-utils

Enable php 7.4 repository 

    sudo yum-config-manager --enable remi-php74
Now run the following command.

    sudo yum install php  php-cli php-fpm php-mysqlnd php-zip php-devel php-gd php-mcrypt php-mbstring php-curl php-xml php-pear php-bcmath php-json
You can add or remove extensions depending on your requirements. 
Restart apache/ httpd server 

     service httpd restart
### Install phpmyAdmin 
Run this command to install latest phpmyadmin 

    yum --enablerepo=remi install phpmyadmin

If you try to access phpmyAdmin by *serveripaddress/phpmyadmin* 
You will see some error. We need to allow access from outside. 
Edit phpmyadmin configuration file by running following command 

    sudo nano /etc/httpd/conf.d/phpMyAdmin.conf
    
Now change the following block by *<Directory /usr/share/phpMyAdmin/>*

      <Directory /usr/share/phpMyAdmin/>
       AddDefaultCharset UTF-8
       Require all granted
      </Directory>
	
Now restart apache server 

    sudo systemctl restart httpd.service
Now you can login remotely. Visit *serveripaddress/phpmyadmin* 
If you can't login using username and password of mysql, update your user's native password by running 

    mysql -uroot -p 
    
    ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY '123456';

### Test Simple PHP Script 
Run the following command 

    nano /var/www/html/test/index.php
Add the following lines in the *index.php* file 

    <?php 
    phpinfo();
    ?>
To access this you need to tell apache server the virtual host configuration. 
Now create a test configuration file on **/etc/httpd/conf.d/** directory 
Create **test.conf** and Add the following lines.

    <VirtualHost *:80>
           ServerName youdomain.com
           DocumentRoot /var/www/html/test
           <Directory /var/www/html/test>
                  AllowOverride All
           </Directory>
    </VirtualHost>
Now restart apache server 

    sudo systemctl restart httpd.service

Now try to access `youdomain.com` if you have configured your Domain's DNS's **blank**  and **www** A record pointing to your server ip. 
Congratulations you have succesfully configured php. Now to test mysql connection edit the /var/www/html/test/index.php file 

      <?php
    $mysqli = new mysqli("localhost","root","123456");
    
    // Check connection
    if ($mysqli -> connect_errno) {
      echo "Failed to connect to MySQL: " . $mysqli -> connect_error;
      exit();
    }
    else {
    echo "connected";
    }
    ?> 
Now save and reload the website. If you see **connected**, then you are good to go.

# # Deploy Laravel Application 
Now its time to deploy our laravel application. First we will install composer and git. 
### Install composer 

    curl -sS https://getcomposer.org/installer | php
    mv composer.phar /usr/bin/composer
    chmod +x /usr/bin/composer
### Install git 

    sudo yum -y install git
Now you can clone or upload your web applications file on /var/www/html directory in appropriate folder. 
For example we will simply just create an application to test login registration functionality. 

    composer create-project --prefer-dist laravel/laravel testapp "5.8.*"
#### Set file permission 

    chown -R apache.apache /var/www/html/testapp 
    chmod -R 755 /var/www/html/testapp 
    chmod -R 755 /var/www/html/testapp/storage
    chcon -R -t httpd_sys_rw_content_t /var/www/html/testapp/storage

Configure the .env file with database credentials. Run migrations 

    php artisan migrate 
To access using your domain , edit the test.conf file /etc/httpd/conf.d directory. You can add more conf files too.

    <VirtualHost *:80>
           ServerName yourdomain.com
           DocumentRoot /var/www/html/testapp/public
    
           <Directory /var/www/html/testapp>
                  AllowOverride All
           </Directory>
    </VirtualHost>
Here we added the document root which is the root directory of your laravel project's root directory. 
Now restart apache server 

    service httpd restart 
Now if you try to access the application by your domain you can access, But when trying to registering, you may get an **sql permission denied** error. To solve this run following command. 

    sudo -P setsebool httpd_can_network_connect_db 1
    
It will enable apache to connect with database. 

# Enjoy ! Keep eyes on this doc. 
### Future Updates 
- Configure Cloudflare SSL certificate
- Configure  SSH 
- Change phpmyadmin address 
