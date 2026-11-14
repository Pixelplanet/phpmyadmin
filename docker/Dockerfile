# phpMyAdmin image for this fork.
#
# Stage "release" packages the checked out commit with phpMyAdmin's own release
# script (bin/internal/create-release.sh), the same way the official release
# archives are made: documentation, translations, production Composer and Yarn
# dependencies, test suite run, development files removed.
#
# The final stage follows the official Apache image (github.com/phpmyadmin/docker,
# apache/Dockerfile at 4f7341b) with the same extensions, PHP settings,
# environment variables, configuration and entrypoint. The differences are that
# the release archive comes from the first stage instead of files.phpmyadmin.net,
# and that phpMyAdmin 6 serves only its public/ folder.
#
# Only committed files end up in the image: the release script builds from a git
# worktree of HEAD, uncommitted changes in the build context are ignored.
#
# Build from the repository root:
#   docker build -f docker/Dockerfile -t phpmyadmin-fork .

FROM php:8.3-cli AS release

RUN set -ex; \
    apt-get update; \
    apt-get install -y --no-install-recommends \
        git \
        gettext \
        make \
        python3-sphinx \
        unzip \
        xz-utils \
        libfreetype6-dev \
        libjpeg-dev \
        libpng-dev \
        libzip-dev \
    ; \
    docker-php-ext-configure gd --with-freetype --with-jpeg; \
    docker-php-ext-install -j "$(nproc)" gd mysqli zip; \
    rm -rf /var/lib/apt/lists/*; \
    echo 'memory_limit=-1' > "$PHP_INI_DIR/conf.d/memory.ini"

COPY --from=composer:2 /usr/bin/composer /usr/bin/composer
COPY --from=node:24-bookworm-slim /usr/local/bin/node /usr/local/bin/node
COPY --from=node:24-bookworm-slim /usr/local/lib/node_modules /usr/local/lib/node_modules
RUN set -ex; \
    ln -s /usr/local/lib/node_modules/npm/bin/npm-cli.js /usr/local/bin/npm; \
    ln -s /usr/local/lib/node_modules/corepack/dist/corepack.js /usr/local/bin/corepack; \
    corepack enable; \
    node --version

WORKDIR /src
COPY . /src

ARG PMA_VERSION=6.0.0-dev
RUN set -ex; \
    git config --global --add safe.directory '*'; \
    git config --global user.email 'build@localhost'; \
    git config --global user.name 'Image build'; \
    git checkout -q --detach; \
    git branch -f image-build HEAD; \
    echo "Building $(git log -1 --format='%h %s')"; \
    export COREPACK_ENABLE_DOWNLOAD_PROMPT=0; \
    kit="release/phpMyAdmin-$PMA_VERSION-all-languages.tar.xz"; \
    # The script ends with `ls -la "$SIGN_FILES"`, which fails because the list starts with a space.
    # The checksum file is written right before that, after the tests passed and the archive was made.
    echo y | ./bin/internal/create-release.sh "$PMA_VERSION" image-build \
        --no-sign --revision-info --kits all-languages --compressions txz \
        || test -s "$kit.sha256"; \
    (cd release && sha256sum -c "phpMyAdmin-$PMA_VERSION-all-languages.tar.xz.sha256"); \
    mv "$kit" /phpMyAdmin.tar.xz

FROM php:8.3-apache

ENV UPLOAD_PROGRESS_EXT_URL="https://github.com/php/pecl-php-uploadprogress/archive/refs/tags/uploadprogress-2.0.2.tar.gz"
ENV UPLOAD_PROGRESS_SHA256="fe3f6cdfcedad563c970c4fd1cda31e422cfc0df5cc9a217d8c80ed3c8d137f5"

# Install dependencies
RUN set -ex; \
    \
    apt-get update; \
    apt-get install -y --no-install-recommends \
        gnupg \
        dirmngr \
    ; \
    \
    savedAptMark="$(apt-mark showmanual)"; \
    \
    apt-get install -y --no-install-recommends \
        libbz2-dev \
        libfreetype6-dev \
        libjpeg-dev \
        libpng-dev \
        libwebp-dev \
        libxpm-dev \
        libzip-dev \
    ; \
    \
    mkdir -p /tmp/uploadprogress; \
    curl -fsSL -o /tmp/uploadprogress/uploadprogress.tar.gz "$UPLOAD_PROGRESS_EXT_URL"; \
    echo "$UPLOAD_PROGRESS_SHA256 /tmp/uploadprogress/uploadprogress.tar.gz" | sha256sum -c -; \
    tar -xf /tmp/uploadprogress/uploadprogress.tar.gz -C /tmp/uploadprogress --strip-components=1; \
    \
    docker-php-ext-configure gd --with-freetype --with-jpeg --with-webp --with-xpm; \
    docker-php-ext-install -j "$(nproc)" \
        bz2 \
        gd \
        mysqli \
        opcache \
        zip \
        bcmath \
        /tmp/uploadprogress \
    ; \
    \
    rm -r /tmp/uploadprogress; \
    \
    apt-mark auto '.*' > /dev/null; \
    apt-mark manual $savedAptMark; \
    extdir="$(php -r 'echo ini_get("extension_dir");')"; \
    ldd "$extdir"/*.so \
        | awk '/=>/ { so = $(NF-1); if (index(so, "/usr/local/") == 1) { next }; gsub("^/(usr/)?", "", so); print so }' \
        | sort -u \
        | xargs -r dpkg-query -S \
        | cut -d: -f1 \
        | sort -u \
        | xargs -rt apt-mark manual; \
    \
    # start: Apache specific build
    a2enmod remoteip; \
    # end: Apache specific build
    \
    apt-get purge -y --auto-remove -o APT::AutoRemove::RecommendsImportant=false; \
    rm -rf /var/lib/apt/lists/*; \
    ldd "$extdir"/*.so | grep -qzv "=> not found" || (echo "Sanity check failed: missing libraries:"; ldd "$extdir"/*.so | grep " => not found"; exit 1); \
    ldd "$extdir"/*.so | grep -q "libzip.so.* => .*/libzip.so.*" || (echo "Sanity check failed: libzip.so is not referenced"; ldd "$extdir"/*.so; exit 1); \
    err="$(php --version 3>&1 1>&2 2>&3)"; \
    [ -z "$err" ] || (echo "Sanity check failed: php returned errors; $err"; exit 1;);

# set recommended PHP.ini settings
# see https://secure.php.net/manual/en/opcache.installation.php
ENV PMA_SSL_DIR=/etc/phpmyadmin/ssl
ENV MAX_EXECUTION_TIME=600
ENV MEMORY_LIMIT=512M
ENV UPLOAD_LIMIT=2048K
ENV TZ=UTC
ENV SESSION_SAVE_PATH=/sessions
RUN set -ex; \
    mkdir $SESSION_SAVE_PATH; \
    mkdir -p $PMA_SSL_DIR; \
    chmod 1777 $SESSION_SAVE_PATH; \
    chmod 755 $PMA_SSL_DIR; \
    chown www-data:www-data /etc/phpmyadmin; \
    chown www-data:www-data $PMA_SSL_DIR; \
    chown www-data:www-data $SESSION_SAVE_PATH; \
    \
    { \
        echo 'opcache.memory_consumption=128'; \
        echo 'opcache.interned_strings_buffer=8'; \
        echo 'opcache.max_accelerated_files=4000'; \
        echo 'opcache.revalidate_freq=2'; \
        echo 'opcache.fast_shutdown=1'; \
    } > $PHP_INI_DIR/conf.d/opcache-recommended.ini; \
    \
    { \
        echo 'session.cookie_httponly=1'; \
        echo 'session.use_strict_mode=1'; \
    } > $PHP_INI_DIR/conf.d/session-strict.ini; \
    \
    { \
        echo 'allow_url_fopen=Off'; \
        echo 'max_execution_time=${MAX_EXECUTION_TIME}'; \
        echo 'max_input_vars=10000'; \
        echo 'memory_limit=${MEMORY_LIMIT}'; \
        echo 'post_max_size=${UPLOAD_LIMIT}'; \
        echo 'upload_max_filesize=${UPLOAD_LIMIT}'; \
        echo 'date.timezone=${TZ}'; \
        echo 'session.save_path=${SESSION_SAVE_PATH}'; \
    } > $PHP_INI_DIR/conf.d/phpmyadmin-misc.ini

# phpMyAdmin 6 keeps its code outside of the web root and serves only public/.
ENV APACHE_DOCUMENT_ROOT=/var/www/phpmyadmin/public
RUN set -ex; \
    mkdir /var/www/phpmyadmin; \
    chown www-data:www-data /var/www/phpmyadmin; \
    sed -ri -e 's!/var/www/html!${APACHE_DOCUMENT_ROOT}!g' /etc/apache2/sites-available/*.conf; \
    sed -ri -e 's!/var/www/!${APACHE_DOCUMENT_ROOT}!g' /etc/apache2/apache2.conf /etc/apache2/conf-available/*.conf

USER www-data:www-data

ARG PMA_VERSION=6.0.0-dev
LABEL org.opencontainers.image.title="phpMyAdmin (fork)" \
    org.opencontainers.image.description="phpMyAdmin built from this fork, packaged like the official Apache image." \
    org.opencontainers.image.licenses="GPL-2.0-only" \
    org.opencontainers.image.version="${PMA_VERSION}" \
    org.opencontainers.image.documentation="https://github.com/phpmyadmin/docker#readme"

# Extract the release archive built in the first stage
COPY --from=release --chown=www-data:www-data /phpMyAdmin.tar.xz /tmp/phpMyAdmin.tar.xz
RUN set -ex; \
    tar -xf /tmp/phpMyAdmin.tar.xz -C /var/www/phpmyadmin --strip-components=1; \
    rm /tmp/phpMyAdmin.tar.xz; \
    mkdir -p /var/www/phpmyadmin/tmp; \
    rm -r -v /var/www/phpmyadmin/public/setup/ /var/www/phpmyadmin/CONTRIBUTING.md /var/www/phpmyadmin/docker/; \
    rm -f -v /var/www/phpmyadmin/.dockerignore; \
    grep -q -F "'configFile' => ROOT_PATH . 'config.inc.php'," /var/www/phpmyadmin/app/vendor_config.php; \
    sed -i "s@'configFile' => .*@'configFile' => '/etc/phpmyadmin/config.inc.php',@" /var/www/phpmyadmin/app/vendor_config.php; \
    grep -q -F "'configFile' => '/etc/phpmyadmin/config.inc.php'," /var/www/phpmyadmin/app/vendor_config.php; \
    php -l /var/www/phpmyadmin/app/vendor_config.php; \
    find /var/www/phpmyadmin -type d -exec chmod 555 {} \;; \
    find /var/www/phpmyadmin -type f -exec chmod 444 {} \;; \
    chmod 1777 /var/www/phpmyadmin/tmp;

# Copy configuration
COPY --chown=www-data:www-data docker/config.inc.php /etc/phpmyadmin/config.inc.php
COPY --chown=www-data:www-data docker/helpers.php /etc/phpmyadmin/helpers.php

# Copy main script (line endings fixed in case the checkout converted them)
COPY docker/docker-entrypoint.sh /docker-entrypoint.sh

USER root
RUN sed -i 's/\r$//' /docker-entrypoint.sh; chmod 755 /docker-entrypoint.sh
ENTRYPOINT [ "/docker-entrypoint.sh" ]
CMD ["apache2-foreground"]
