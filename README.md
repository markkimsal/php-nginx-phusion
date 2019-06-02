# https://github.com/markkimsal/php-nginx-phusion

## https://hub.docker.com/r/markkimsal/php-nginx-phusion

## Forked from: [the Docker Community](https://github.com/docker-library/php)


You absolutely need both nginx and php-fpm in the same container if you're at all interested in deploying to a cluster or swarm. Docker swarm's volumes are not cluster/swarm aware. It's complicated, but if you're here, you probably ran into the same problem.

Next problem is, how do you get nginx onto the official php docker image? Sure, apk add or apt get, but it's not running, and the entrypoint script is not capable of running two services.

Phusion base image supplies runit and /sbin/my\_init to run multiple services.

##PHP extension installation

You can install extension just like the "official" php docker container

```
FROM markkimsal/php-nginx-phusion:7.3-fpm

RUN docker-php-ext-install mysqli pdo_mysql \
    && docker-php-ext-enable mysqli pdo_mysql
```


##Nginx Vhost Sample

Pass PHP requests to localhost 9000.  This assumes everything is copied or mounted to `/app`

```
map $http_x_forwarded_proto $fastcgi_param_https_variable {
  default '';
  https 'on';
}

server {
  listen 0.0.0.0:8080 default;
  server_name _;

  root /app/public;
  index index.php index.html index.htm;

  location / {
    try_files $uri $uri/ /index.php?$args;
    #try_files $uri $uri/index.php;
  }

  location ~ \.php$ {
    fastcgi_pass localhost:9000;
    include fastcgi.conf;

    fastcgi_index /index.php;

    fastcgi_param HTTPS $fastcgi_param_https_variable;
    fastcgi_param PATH_INFO $fastcgi_path_info;
    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    fastcgi_param SCRIPT_NAME $fastcgi_script_name;
  }
}
```

##MSSQL Sqlsrv extension
Mssql gives .deb packages for installing ODBC and sqlsrv extension on PHP.  This means you need Debian/Ubuntu instead of alpine.
```
FROM markkimsal/php-nginx-phusion:7.3-fpm

# Add Microsoft repo for Microsoft ODBC Driver 13 for Linux
RUN apt-get update \
    && apt-get install -y apt-utils gnupg curl build-essential libaio1 software-properties-common locales iputils-ping --no-install-recommends \
    && apt-get install -y \
    apt-transport-https \
    && curl https://packages.microsoft.com/keys/microsoft.asc | apt-key add - \
    && curl https://packages.microsoft.com/config/debian/9/prod.list > /etc/apt/sources.list.d/mssql-release.list \
    && apt-get update

# Install Dependencies
RUN ACCEPT_EULA=Y apt-get install -y \
    unixodbc \
    unixodbc-dev \
    libgss3 \
    odbcinst \
    msodbcsql17 \
    locales \
    && echo "en_US.UTF-8 UTF-8" > /etc/locale.gen && locale-gen

# Install pdo_sqlsrv and sqlsrv from PECL. Replace pdo_sqlsrv-4.1.8preview with preferred version.
RUN pecl install pdo_sqlsrv sqlsrv \
    && docker-php-ext-enable pdo_sqlsrv sqlsrv
```
