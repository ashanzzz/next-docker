FROM nextcloud:fpm-alpine
########################
# 1. 切回 root，用来重建 www-data（99:100）
########################
USER root

RUN set -ex; \
    # 把 www-data 的 GID 改为 100
    sed -i 's/^www-data:x:[0-9]\+:/www-data:x:100:/' /etc/group; \
    # 把 www-data 的 UID 改为 99，GID 改为 100
    sed -i 's/^www-data:x:[0-9]\+:[0-9]\+:/www-data:x:99:100:/' /etc/passwd; \
    # 确保这些目录存在
    mkdir -p /var/www/html \
             /usr/local/etc/php \
             /var/log/supervisord \
             /var/run/supervisord; \
    # 修改属主
    chown -R www-data:www-data \
          /var/www/html \
          /usr/local/etc/php \
          /var/log/supervisord \
          /var/run/supervisord

########################
# 2. 安装运行时依赖
########################
# 指定要安装的 glibc 版本
ARG GLIBC_VER=2.36-r2
ENV GLIBC_VER=${GLIBC_VER}
RUN set -ex; \
    # 1. 安装必要工具 & 导入 sgerrand 公钥
    apk add --no-cache --virtual .build-deps \
      ca-certificates wget && \
    wget -q -O /etc/apk/keys/sgerrand.rsa.pub \
      https://alpine-pkgs.sgerrand.com/sgerrand.rsa.pub && \
    \
    # 2. 下载并安装 glibc 包
    wget -q \
      https://github.com/sgerrand/alpine-pkg-glibc/releases/download/${GLIBC_VER}/glibc-${GLIBC_VER}.apk && \
    apk add --no-cache glibc-${GLIBC_VER}.apk && \
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
