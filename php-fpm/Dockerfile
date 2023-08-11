FROM php:fpm-alpine

COPY ./html /var/www/html
RUN chown -R www-data:www-data /var/www/html

#non-root実行で試したい場合は以下をコメントアウト
#RUN adduser -D www-user
#USER www-user
