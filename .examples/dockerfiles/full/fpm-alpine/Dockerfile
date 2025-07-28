FROM nextcloud:fpm-alpine
########################
# 1. 切回 root，用来重建 www-data（99:100）
########################
USER root
# Alpine BusyBox 自带 adduser/deluser，重建 www-data
RUN set -ex; \
    deluser www-data; \
    delgroup www-data; \
    addgroup -g 100 www-data; \
    adduser -u 99 -G www-data -h /var/www/html -D -S -s /bin/sh www-data; \
    # 把 Nextcloud 程序目录和 PHP 配置目录都给改一下属主
    chown -R www-data:www-data /var/www/html /usr/local/etc/php

########################
# 2. 安装运行时依赖
########################
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
########################
# 3. 编译扩展、安装 SMB 等
########################
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

########################
# 4. Supervisord 目录也要给权限
########################
RUN set -ex; \
    mkdir -p /var/log/supervisord /var/run/supervisord; \
    chown -R www-data:www-data /var/log/supervisord /var/run/supervisord

########################
# 5. 复制你的 supervisord.conf，并且在其中加上 user=www-data
########################
COPY supervisord.conf /
# 在 supervisord.conf 的 [program:cron] 段落内加上一行：
#   user=www-data

########################
# 6. 切回 www-data:99:100，后续 entrypoint + supervisord 都以这个用户运行
########################
USER www-data:www-data

ENV NEXTCLOUD_UPDATE=1

CMD [ "/usr/bin/supervisord", "-c", "/supervisord.conf" ]
