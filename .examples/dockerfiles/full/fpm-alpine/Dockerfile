FROM nextcloud:fpm-alpine
# 切回 root，调整 www-data 用户为 99:100
USER root
# Alpine BusyBox 自带 adduser/deluser，重建 www-data
RUN set -ex; \
    deluser www-data; \
    delgroup www-data; \
    addgroup -g 100 www-data; \
    adduser -u 99 -G www-data -h /var/www/html -D -S -s /bin/sh www-data; \
    # 把 Nextcloud 程序目录和 PHP 配置目录都给改一下属主
    chown -R www-data:www-data /var/www/html /usr/local/etc/php

RUN set -ex; \
    \
    apk add --no-cache \
        ffmpeg \
        imagemagick \
        procps \
        samba-client \
        supervisor \
#       libreoffice \
    ;

RUN set -ex; \
    \
    apk add --no-cache --virtual .build-deps \
        $PHPIZE_DEPS \
        imap-dev \
        krb5-dev \
        openssl-dev \
        samba-dev \
        bzip2-dev \
    ; \
    \
    docker-php-ext-configure imap --with-kerberos --with-imap-ssl; \
    docker-php-ext-install \
        bz2 \
        imap \
    ; \
    pecl install smbclient; \
    docker-php-ext-enable smbclient; \
    \
    runDeps="$( \
        scanelf --needed --nobanner --format '%n#p' --recursive /usr/local/lib/php/extensions \
            | tr ',' '\n' \
            | sort -u \
            | awk 'system("[ -e /usr/local/lib/" $1 " ]") == 0 { next } { print "so:" $1 }' \
    )"; \
    apk add --virtual .nextcloud-phpext-rundeps $runDeps; \
    apk del .build-deps

# Supervisord 目录、锁文件也要 www-data 拥有
RUN set -ex; \
    mkdir -p /var/log/supervisord /var/run/supervisord; \
    chown -R www-data:www-data /var/log/supervisord /var/run/supervisord


COPY supervisord.conf /

ENV NEXTCLOUD_UPDATE=1

CMD ["/usr/bin/supervisord", "-c", "/supervisord.conf"]
