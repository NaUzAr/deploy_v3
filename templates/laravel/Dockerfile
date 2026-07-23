FROM php:8.3-fpm-alpine

RUN apk add --no-cache nginx supervisor libpng-dev libzip-dev postgresql-dev nodejs npm \
    && docker-php-ext-install pdo pdo_pgsql zip gd bcmath pcntl

COPY --from=composer:latest /usr/bin/composer /usr/bin/composer
COPY nginx.conf /etc/nginx/http.d/default.conf
COPY supervisord.ini /etc/supervisor.d/supervisord.ini

WORKDIR /var/www/html
COPY src/ /var/www/html/
RUN rm -rf vendor composer.lock && composer install --optimize-autoloader --no-dev --no-interaction \
    && npm install && npm run build
RUN chown -R www-data:www-data /var/www/html \
    && chmod -R 755 /var/www/html/storage /var/www/html/bootstrap/cache

EXPOSE 80 8080
CMD ["/usr/bin/supervisord", "-c", "/etc/supervisord.conf"]
