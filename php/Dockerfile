FROM php:8.3.36-apache
RUN docker-php-ext-install mysqli && docker-php-ext-enable mysqli
RUN apt-get update && apt-get upgrade -y
# Inside your Dockerfile
RUN echo "post_max_size = 32M" >> /usr/local/etc/php/conf.d/custom.ini \
    && echo "upload_max_filesize = 32M" >> /usr/local/etc/php/conf.d/custom.ini
