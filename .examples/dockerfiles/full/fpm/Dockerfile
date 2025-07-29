FROM nextcloud:fpm
################################################
# 1. 切回 root → 把 www-data 改成 99:100
################################################
USER root

RUN set -ex; \
    # 👉 改 GID → 100
    sed -i 's/^www-data:x:[0-9]\+:/www-data:x:100:/' /etc/group; \
    # 👉 改 UID → 99，GID → 100
    sed -i 's/^www-data:x:[0-9]\+:[0-9]\+:/www-data:x:99:100:/' /etc/passwd; \
    \
    # 确保关键目录存在并归属 www-data
    mkdir -p /var/www/html \
             /usr/local/etc/php \
             /var/log/supervisord \
             /var/run/supervisord; \
    chown -R www-data:www-data \
          /var/www/html \
          /usr/local/etc/php \
          /var/log/supervisord \
          /var/run/supervisord
################################################
# 2. 安装运行时依赖（保持官方原配方）
################################################
RUN set -ex; \
    \
    apt-get update; \
    apt-get install -y --no-install-recommends \
        ffmpeg \
        ghostscript \
        libmagickcore-6.q16-6-extra \
        procps \
        smbclient \
        supervisor \
#       libreoffice \
    ; \
    rm -rf /var/lib/apt/lists/*
################################################
# 3. 编译 PHP 扩展（imap、smbclient）
################################################
RUN set -ex; \
    \
    savedAptMark="$(apt-mark showmanual)"; \
    \
    apt-get update; \
    apt-get install -y --no-install-recommends \
        libbz2-dev \
        libc-client-dev \
        libkrb5-dev \
        libsmbclient-dev \
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
# reset apt-mark's "manual" list so that "purge --auto-remove" will remove all build dependencies
    apt-mark auto '.*' > /dev/null; \
    apt-mark manual $savedAptMark; \
    ldd "$(php -r 'echo ini_get("extension_dir");')"/*.so \
        | awk '/=>/ { so = $(NF-1); if (index(so, "/usr/local/") == 1) { next }; gsub("^/(usr/)?", "", so); print so }' \
        | sort -u \
        | xargs -r dpkg-query --search \
        | cut -d: -f1 \
        | sort -u \
        | xargs -rt apt-mark manual; \
    \
    apt-get purge -y --auto-remove -o APT::AutoRemove::RecommendsImportant=false; \
    rm -rf /var/lib/apt/lists/*

################################################
# 4. Supervisord 目录再保险
################################################
RUN mkdir -p /var/log/supervisord /var/run/supervisord && \
    chown -R www-data:www-data /var/log/supervisord /var/run/supervisord

################################################
# 5. 复制 supervisord.conf（务必在 program:cron 段加入 user=www-data）
################################################


COPY supervisord.conf /

################################################
# 6. 切回 www-data 用户 & 默认启动
################################################
USER www-data:www-data

ENV NEXTCLOUD_UPDATE=1
CMD ["/usr/bin/supervisord", "-c", "/supervisord.conf"]
