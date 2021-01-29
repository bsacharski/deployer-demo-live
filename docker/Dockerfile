FROM php:7.4-cli-alpine AS composer
RUN apk add wget
COPY ./scripts/install-composer.sh /tmp/install-composer.sh
RUN sh /tmp/install-composer.sh





FROM php:7.4-cli-alpine AS deployer
RUN apk add \
        git \
        openssh-client \
        rsync

RUN ssh-keygen \
    -q \
    -b 2048 \
    -t rsa \
    -f ~/.ssh/id_rsa

RUN git config --global user.email "demo@deployer.test" \
    && git config --global user.name "DEMO Deployer"

ARG XDEBUG_VERSION=2.9.8
RUN set -eux; \
	apk add --no-cache --virtual .build-deps $PHPIZE_DEPS; \
	pecl install xdebug-$XDEBUG_VERSION; \
	docker-php-ext-enable xdebug; \
	apk del .build-deps

COPY --from=composer /tmp/composer /bin/composer

VOLUME [ "/project" ]
WORKDIR /project





FROM php:7.4-apache AS server
RUN apt-get update && apt-get install -y \
        acl \
        git \
        openssh-server \
        rsync \
        sudo \
    && rm -rf /var/lib/apt/lists/*

RUN apt-get update && apt-get install -y \
        libpq-dev \
        libzip-dev \
        unzip \
        zlib1g-dev \
    && docker-php-ext-install \
        pdo \
        pdo_pgsql\
        pgsql \
        zip \
    && rm -rf /var/lib/apt/lists/*

# SSH login fix. Otherwise user is kicked off after login
RUN mkdir /run/sshd \
    && sed -i 's@session\s*required\s*pam_loginuid.so@session optional pam_loginuid.so@g' /etc/pam.d/sshd

# Configure Apache to expose healthcheck & configure site to use /var/www/html/current ad document root
COPY conf/healthcheck.conf /etc/apache2/sites-available/healthcheck.conf
COPY ./initial-site /var/www/html/initial-site

ENV APACHE_DOCUMENT_ROOT /var/www/html/current/public
RUN sed -ri -e 's!/var/www/html!${APACHE_DOCUMENT_ROOT}!g' /etc/apache2/sites-available/000-default.conf \
    && sed -ri -e 's!/var/www/!${APACHE_DOCUMENT_ROOT}!g' /etc/apache2/apache2.conf /etc/apache2/conf-available/*.conf \
    && ln -s /var/www/html/initial-site /var/www/html/current \
    && chown -R www-data:www-data /var/www/html \
    && echo "Listen 81" >> /etc/apache2/ports.conf \
    && a2enmod rewrite \
    && a2ensite healthcheck
    
RUN useradd \
        --create-home \
        deployer \
    && echo 'deployer:deployer' | chpasswd \
    && echo 'deployer ALL=(ALL) ALL' >> /etc/sudoers \
    && mkdir ~deployer/.ssh \
    && touch ~deployer/.ssh/authorized_keys \
    && touch ~deployer/.ssh/known_hosts \
    && chown -R deployer:deployer ~deployer/.ssh \
    && chmod 700 ~deployer/.ssh \
    && chmod 600 ~deployer/.ssh/authorized_keys ~deployer/.ssh/known_hosts

COPY scripts/start-servers.sh /usr/local/bin/start-servers
COPY --from=composer /tmp/composer /usr/local/bin/composer
COPY --from=deployer /root/.ssh/id_rsa /root/.ssh/id_rsa.pub /tmp/

RUN chmod a+x /usr/local/bin/start-servers \
    && cat /tmp/id_rsa.pub >> ~deployer/.ssh/authorized_keys \
    && mv /tmp/id_rsa /tmp/id_rsa.pub ~deployer/.ssh/ \
    && chmod 600 ~deployer/.ssh/id_rsa ~deployer/.ssh/id_rsa.pub \
    && chown deployer:deployer ~deployer/.ssh/id_rsa ~deployer/.ssh/id_rsa.pub

RUN ssh-keyscan -H github.com >> ~deployer/.ssh/known_hosts

EXPOSE 22 80 81
CMD [ "start-servers" ]
