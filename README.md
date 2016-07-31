# install-owncloud-nginx-php7-postgresql
A tutorial for installing owncloud on FreeBSD. This is a rough draft and is not fully functional yet.

## Install Owncloud on FreeBSD with Nginx, PostgreSQL, PHP-fpm.



Install PostgreSQL packages:

`pkg install postgresql93-server postgresql93-client`

After PostgreSQL has completed installation we need to enable it to start at boot time. Enter:

`sysrc postgresql_enable=YES` 

Or:

`echo 'postgresql_enable="YES"' >> /etc/rc.conf` 


Everything look Good? Ok, we now must initialize our PostgreSQL database. Enter:

`service postgresql initdb` 

Lastly, start your new database by entering: 

`service postgresql start`

If there were no errors after this last step then you're good to go! A database is worthless without some configurations. Let's configure stuff!

As sudo/root, enter the following:

    passwd pgsql
    Changing local password for pgsql
    New Password: test5678
    Retype New Password: test5678

At this point your FreeBSD server has a new user called "pgsql" with a password of "test5678". *_Hopefully_* you chose a better password for your pgsql user. 

Login to pgsql with the following command:

`su pgsql`

Time to put user pgsql to work! Make sure user pgsql is in its home directory by entering: 

`cd /usr/local/pgsql`

Let's get down to business. User _pgsql_ needs to setup a *ROLE* and create a *database* for owncloud.

As pgsql user enter:

    createdb owncloud
    psql owncloud
    
In _psql_ you will be greeted with the following:

     psql (9.3.13)
     Type "help" for help.

     owncloud=#


    owncloud=# CREATE ROLE owncloud;
    CREATE ROLE
    owncloud=# ALTER ROLE owncloud WITH PASSWORD 'oc5678';
    ALTER ROLE
    owncloud=# ALTER ROLE owncloud WITH LOGIN;
    ALTER ROLE
    owncloud=# ALTER DATABASE owncloud OWNER TO owncloud;
    ALTER DATABASE
    owncloud=# \list
                               List of databases
       Name    |  Owner   | Encoding | Collate | Ctype | Access privileges
    -----------+----------+----------+---------+-------+-------------------
     owncloud  | owncloud | UTF8     | C       | C     |
     postgres  | pgsql    | UTF8     | C       | C     |
     template0 | pgsql    | UTF8     | C       | C     | =c/pgsql         +
               |          |          |         |       | pgsql=CTc/pgsql
     template1 | pgsql    | UTF8     | C       | C     | =c/pgsql         +
               |          |          |         |       | pgsql=CTc/pgsql
    
    owncloud=# \q

--------------------

Install PHP 7

`pkg install php70`

Additional PHP libraries to install:

`pkg install php70-curl php70-mcrypt php70-zip php70-xmlrpc php70-xml php70-imap php70-json php70-pgsql php70-opcache php70-gmp php70-gd php70-ldap php70-intl php70-openssl php70-exif php70-extensions php70-pdo_pgsql php70-bz2 php70-zlib php70-fileinfo php70-mbstring`

Configure PHP-FPM

`vim /usr/local/etc/php-fpm.conf`

Change the lines to be:

    listen = /var/run/php-fpm.sock
    ...
    ...
    listen.owner = www
    listen.group = www
    listen.mode = 0660

Save and close the file.

Enable and start PHP-FPM service with the following commands:

`sysrc php_fpm_enable=YES`

Or:

`echo 'php_fpm_enable="YES' >> /etc/rc.conf`

`service php-fpm start`

----------------------

If you're installing owncloud for testing purposes and wish to add a self-signed SSL certificate you can do so with these steps:

`mkdir -p /usr/local/etc/self-cert/`

`cd /usr/local/etc/self-cert/`

`openssl req -new -x509 -days 365 -nodes -out /usr/local/etc/self-cert/owncloud.crt -keyout /usr/local/etc/self-cert/owncloud.key`

Now, change certificate permissions to prevent the wrong people from accessing your certs:

`chmod 600 * `


Using the command `ls -la` should list your certificate permissions. They will look something like this:

    root@spork:/usr/local/etc/self-cert # ls -la
    total 18
    drwxr-xr-x   2 root  wheel    4 Jul 30 19:15 .
    drwxr-xr-x  22 root  wheel   37 Jul 30 19:14 ..
    -rw-------   1 root  wheel  875 Jul 30 19:15 owncloud.crt
    -rw-------   1 root  wheel  916 Jul 30 19:15 owncloud.key

-----------------


Download ownCloud:

`cd /tmp`

`wget https://download.owncloud.org/community/owncloud-9.1.0.tar.bz2`

Note: FreeBSD does not come with **wget** pre-installed. If your system does not have wget you can install it with: `pkg install wget`.


We will verify the SHA256 sum by downloading either of the corresponding files:

`wget https://download.owncloud.org/community/owncloud-9.1.0.tar.bz2.sha256`

`wget https://download.owncloud.org/community/owncloud-9.1.0.zip.sha256`

Run the appropriate command based on archive and sum check type:


    sha256 -c owncloud-9.1.0.tar.bz2.sha256 -s owncloud-9.1.0.tar.bz2
    sha256 -c owncloud-9.1.0.zip.sha256 -s owncloud-9.1.0.zip

You should also verify the GPG signature. FreeBSD does not includ GPG by default so you will have to add the package via: `pkg install gnupg`

    wget https://download.owncloud.org/community/owncloud-9.1.0.tar.bz2.asc
    wget https://owncloud.org/owncloud.asc
    gpg --import owncloud.asc
    gpg --verify owncloud-9.1.0.tar.bz2.asc owncloud-9.1.0.tar.bz2


Everything looks good? Great! Next, let's extract the archive contents. Depending on which type of owncloud download archive chose, use either of the following commands:
   
`tar -xjf owncloud-9.1.0.tar.bz2`

Or:

`unzip owncloud-9.1.0.zip` ## Note: You must have the package _unzip_ installed to unzip a .zip archive. Install with: `pkg install unzip`

Now that you've unpacked or unzipped your archive in `/tmp/`, we need to move our owncloud directory to its final destination:

Create your owncloud webserver root path:

`mkdir -p /srv/www/owncloud`

Copy the owncloud directory from /tmp/ to your newly created webroot:

`cp -r owncloud/* /srv/www/owncloud/`

Change ownership of your owncloud directory:

`chown -R www:www /srv/www/owncloud/`

Set permissions of your owncloud directory:

`chmod -R 0750 /srv/www/owncloud`

-------------------------

Install nginx package:

`pkg install nginx`

After installation, enter: 

`echo 'nginx_enable="YES"' >> /etc/rc.conf` 

This will enable Nginx to start automatically on your server during boot time. Let's setup nginx as our webserver for owncloud. For this example because we used self-signed keys, nginx is going to be configured to use SSL. First thing's first -make a backup of nginx.conf. 

    cd /usr/local/etc/nginx/
    cp nginx.conf nginx.conf.bkup

Now, edit nginx.conf:

`vim nginx.conf`

Enter the following:
	
	load_module /usr/local/libexec/nginx/ngx_mail_module.so;
	load_module /usr/local/libexec/nginx/ngx_stream_module.so;
	
	#user  nobody;
	worker_processes  1;
	
	#error_log  logs/error.log;
	#error_log  logs/error.log  notice;
	#error_log  logs/error.log  info;
	
	#pid        logs/nginx.pid;
	
	
	events {
	    worker_connections  1024;
	}	
	
		
	http {
	    include       mime.types;
	    default_type  application/octet-stream;
	
	    #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
	    #                  '$status $body_bytes_sent "$http_referer" '
	    #                  '"$http_user_agent" "$http_x_forwarded_for"';
	
	    #access_log  logs/access.log  main;
	
	    sendfile        on;
	    #tcp_nopush     on;
	
	    #keepalive_timeout  0;
	    keepalive_timeout  65;
	
	    #gzip  on;
	
	  include /usr/local/etc/nginx/sites-enabled/*;
	
	}

Save and close. Now we must add our owncloud nginx server block. Enter:

`mkdir -p /usr/local/etc/nginx/sites-available sites-enabled`

Create your owncloud nginx server block file:

`vim sites-available/owncloud`

Enter the following:

	upstream php-handler {
	#  server 127.0.0.1:9000;
	   server unix:/var/run/php-fpm.sock;
	}
	
	server {
	  listen 443 ssl;
	  #server_name default_server;
	
	  ssl_certificate /usr/local/etc/self-cert/owncloud.crt;
	  ssl_certificate_key /usr/local/etc/self-cert/owncloud.key;
	
	  # Path to the root of your installation
	  root /srv/www/owncloud/;
	  # set max upload size
	  client_max_body_size 10G;
	  fastcgi_buffers 64 4K;
	
	  # Disable gzip to avoid the removal of the ETag header
	  gzip off;
	
	  # Uncomment if your server is build with the ngx_pagespeed module
	  # This module is currently not supported.
	  #pagespeed off;
	
	  rewrite ^/caldav(.*)$ /remote.php/caldav$1 redirect;
	  rewrite ^/carddav(.*)$ /remote.php/carddav$1 redirect;
	  rewrite ^/webdav(.*)$ /remote.php/webdav$1 redirect;
	
	
	  index index.php;
	  error_page 403 /core/templates/403.php;
	  error_page 404 /core/templates/404.php;
	
	  location = /robots.txt {
	    allow all;
	    log_not_found off;
	    access_log off;
	  }
	
	  location ~ ^/(?:\.htaccess|data|config|db_structure\.xml|README){
	    deny all;
	  }
	
	  location / {
	    # The following 2 rules are only needed with webfinger
	    rewrite ^/.well-known/host-meta /public.php?service=host-meta last;
	    rewrite ^/.well-known/host-meta.json /public.php?service=host-meta-json last;
	
	    rewrite ^/.well-known/carddav /remote.php/carddav/ redirect;
	    rewrite ^/.well-known/caldav /remote.php/caldav/ redirect;
	
	    rewrite ^(/core/doc/[^\/]+/)$ $1/index.html;
	
	    try_files $uri $uri/ =404;
	  }
	
	  location ~ \.php(?:$|/) {
	    fastcgi_split_path_info ^(.+\.php)(/.+)$;
	    include fastcgi_params;
	    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
	    fastcgi_param PATH_INFO $fastcgi_path_info;
	    fastcgi_param HTTPS on;
	    fastcgi_pass php-handler;
	    fastcgi_intercept_errors on;
	  }
	
	 # Adding the cache control header for js and css files
	  # Make sure it is BELOW the location ~ \.php(?:$|/) { block
	  location ~* \.(?:css|js)$ {
	    add_header Cache-Control "public, max-age=7200";
	    # Add headers to serve security related headers
	    add_header Strict-Transport-Security "max-age=15768000; includeSubDomains; preload;";
	    add_header X-Content-Type-Options nosniff;
	    add_header X-Frame-Options "SAMEORIGIN";
	    add_header X-XSS-Protection "1; mode=block";
	    add_header X-Robots-Tag none;
	    # Optional: Don't log access to assets
	    access_log off;
	  }
	
	  # Optional: Don't log access to other assets
	  location ~* \.(?:jpg|jpeg|gif|bmp|ico|png|swf)$ {
	    access_log off;
	  }
	}
		

Save and close. If everything is correct up to this point we can start nginx:

`service start nginx`

If nginx starts successfully, open your web browser and type: `https://<your-server-ip-address>`. Accept the invalid self-signed certificate your browser warns you about and continue. This will take you to owncloud setup page. Create a new user and add your PostgreSQL credentials. Finally click **Finish setup** and enjoy your owncloud! 

	
