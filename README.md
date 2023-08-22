First, create laravel project
[Click this if you not install composer](https://getcomposer.org/download/)
```sh
composer create-project laravel/laravel your-project-name
```


Second, create Dockerfile in your project name, this here is code in Docker file:
```sh
FROM php:8.1-fpm
# set your user name, ex: user=bernardo
ARG user=carlos
ARG uid=1000
# Install system dependencies
RUN apt-get update && apt-get install -y \
    git \
    curl \
    libpng-dev \
    libonig-dev \
    libxml2-dev \
    zip \
    unzip
# Clear cache
RUN apt-get clean && rm -rf /var/lib/apt/lists/*

# Install PHP extensions
RUN docker-php-ext-install pdo_mysql mbstring exif pcntl bcmath gd sockets

# Get latest Composer
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

# Create system user to run Composer and Artisan Commands
RUN useradd -G www-data,root -u $uid -d /home/$user $user
RUN mkdir -p /home/$user/.composer && \
    chown -R $user:$user /home/$user
    
# Set working directory
WORKDIR /var/www

USER $user
```


Next, create folder nginx and create Dockerfile in this folder, Dockerfile in here like this:
```sh
# Use the official Nginx image as the base image
FROM nginx:latest

# Remove the default Nginx configuration file
RUN rm /etc/nginx/conf.d/default.conf

# Copy your custom Nginx configuration file
COPY nginx.conf /etc/nginx/conf.d/
```



In this folder nginx, create file nginx.conf to congfig, in this file: 
```sh
server {
    listen 80;
    index index.php;
    root /var/www/public;

    client_max_body_size 51g;
    client_body_buffer_size 512k;
    client_body_in_file_only clean;

    location ~ \.php$ {
        try_files $uri =404;
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass app:9000;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
        fastcgi_read_timeout 1800; 
    }

    location / {
        try_files $uri $uri/ /index.php?$query_string;
        gzip_static on;
    }

    error_log  /var/log/nginx/error.log;
    access_log /var/log/nginx/access.log;
}
```


Next, in your project folder, create docker-compose.yml:
```sh
version: '3'

services:
#build app image with dockerfile
  app:
    build:
      context: .
      dockerfile: Dockerfile
    restart: unless-stopped
    working_dir: /var/www/
    volumes:
      - ./:/var/www/
    networks:
      - laravel
#build nginx image with dockerfile
  nginx:
    build:
      context: ./nginx
      dockerfile: Dockerfile
    ports:
      - "8888:80"
    volumes:
      - ./:/var/www/
      - ./nginx/:/etc/nginx/conf.d/
    networks:
      - laravel
#build minio image, pull from dockerhub
  minio:
    image: minio/minio
    container_name: minio
    ports:
      - "9000:9000"
      - "9001:9001"
    volumes:
      - ./storage:/data
    environment:
      MINIO_ROOT_USER: thanhvu
      MINIO_ROOT_PASSWORD: zucoder2002
    command: server --console-address ":9001" /data

#build redis image, pull from dockerhub
  redis:
    image: redis:latest
    ports:
      - "6379:6379"
    networks:
      - laravel
#build mysql image, pull from dockerhub
  db:
    image: mysql
    restart: unless-stopped
    environment:
      - MYSQL_ROOT_PASSWORD=zucoder2002
      - MYSQL_DATABASE=laravel
      - MYSQL_USER=
      - MYSQL_PASSWORD=your-password
    ports:
      - "3307:3306"
    volumes:
      - mysql-data:/var/lib/mysql
    networks:
      - laravel

volumes:
  storage: {}
  mysql-data:

networks:
  laravel:
    driver: bridge
```


in terminal project, type this to install:
```sh
docker-compose up -d
```
All container after install

![Screenshot from 2023-08-22 16-36-43](https://github.com/thanhvuclassmethod/docker-laravel-guide/assets/141601458/807811df-9d37-4335-a352-97139fca907d)


If you want to connect laravel with mysql, in env file change
DB_HOST=db
DB_DATABASE=laravel
DB_PASSWORD=your-password


localhost:8888 for project laravel
![Screenshot from 2023-08-22 16-32-22](https://github.com/thanhvuclassmethod/docker-laravel-guide/assets/141601458/6d9bf539-28a7-4595-ac0f-4e71f07ee2c1)

localhost:9001 for minio
![Screenshot from 2023-08-22 16-32-56](https://github.com/thanhvuclassmethod/docker-laravel-guide/assets/141601458/baa1ea9f-a667-4581-80c4-7c7e09043947)








