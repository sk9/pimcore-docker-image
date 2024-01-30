ARG APP_DIR=/var/www/html
ARG FRONTEND_DIR=/var/www/frontend
ARG APP_ENV=prod
ARG APP_DEBUG=false
ARG PIMCORE_ENVIRONMENT=prod
ARG SYMFONY_DEPRECATIONS_HELPER=disabled
ARG NODE_ENV=dev

FROM node:16.17.1-buster AS frontend-build
ARG NODE_ENV
ENV NODE_ENV=$NODE_ENV

RUN mkdir -p /build/frontend
RUN mkdir -p /build/app

WORKDIR /build/frontend
COPY --chown=www-data:www-data ./frontend /build/frontend

RUN yarn && yarn run prod

FROM serversideup/php:8.1-fpm-nginx-v2.2.1

ARG APP_DIR=/var/www/html
ARG FRONTEND_DIR=/var/www/frontend
ARG APP_ENV=prod
ARG APP_DEBUG=false
ARG PIMCORE_ENVIRONMENT=prod
ARG SYMFONY_DEPRECATIONS_HELPER=disabled

ENV APP_DIR=$APP_DIR
ENV FRONTEND_DIR=$FRONTEND_DIR
ENV APP_ENV=$APP_ENV
ENV APP_DEBUG=$APP_DEBUG
ENV PIMCORE_ENVIRONMENT=$PIMCORE_ENVIRONMENT
ENV SYMFONY_DEPRECATIONS_HELPER=$SYMFONY_DEPRECATIONS_HELPER
ENV SSL_MODE="off"
ENV PHP_DATE_TIMEZONE="Europe/Berlin"
ENV PHP_POST_MAX_SIZE="1000M"

#Enable or disable different automations
ENV AUTORUN_LARAVEL_STORAGE_LINK=false
ENV AUTORUN_ENABLED=true
ENV AUTORUN_PIMCORE_MIGRATION=true
ENV AUTORUN_SYMFONY_ASSETS_LINK=true
ENV AUTORUN_SYMFONY_CACHE_WARMUP=true
ENV AUTORUN_PIMCORE_CLASSES_REBUILD=true
ENV DEBUG_OUTPUT=false

WORKDIR $APP_DIR

COPY --chown=9999:9999 ./app $APP_DIR
COPY --chown=9999:9999 --from=frontend-build /build/app/public/dist $APP_DIR/public/dist

# Install Pimcore Recommended Tools using regular Ubuntu commands
RUN apt-get update \
    && apt-get install -y --no-install-recommends libimage-exiftool-perl imagemagick libmagickcore-6.q16-6-extra php8.1-imagick pngquant git optipng libimage-exiftool-perl webp jpegoptim ffmpeg libreoffice libreoffice-script-provider-python libreoffice-math xfonts-75dpi poppler-utils inkscape libxrender1 libfontconfig1 ghostscript patch \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/* /usr/share/doc/*

RUN composer install --prefer-dist --no-dev --optimize-autoloader --no-interaction --no-scripts --quiet

WORKDIR "$APP_DIR"

#make sure no cache artifacts are left over
RUN rm -rf $APP_DIR/var/cache/prod

#application config
COPY --chown=9999:9999 ./deployment/nginx/.htpasswd $APP_DIR/.htpasswd
COPY --chown=9999:9999 ./app/config/ecommerce/config.yaml.live $APP_DIR/config/ecommerce/config.yaml
COPY --chown=9999:9999 ./app/config/dynamic-search.yaml.live $APP_DIR/config/dynamic-search.yaml

#nginx config
COPY ./deployment/nginx/pimcore.conf /etc/nginx/server-opts.d/pimcore.conf
COPY ./deployment/nginx/fastcgi-php.conf /etc/nginx/snippets/fastcgi-php.conf
COPY ./deployment/nginx/site-opts.d/http.conf /etc/nginx/site-opts.d/http.conf
COPY ./deployment/nginx/sites-available/ssl-off /etc/nginx/sites-available/ssl-off
COPY ./deployment/nginx/performance.conf /etc/nginx/server-opts.d/performance.conf
COPY ./deployment/nginx/mime.types /etc/nginx/mime.types

#s6 overlay
COPY --chmod=755 ./deployment/scripts/pimcore-automations /etc/s6-overlay/scripts/pimcore-automations
COPY --chmod=755 ./deployment/scripts/pimcore-maintenance /etc/s6-overlay/scripts/pimcore-maintenance
COPY --chmod=755 ./deployment/config/s6/pimcore-automations /etc/s6-overlay/s6-rc.d/pimcore-automations
COPY --chmod=755 ./deployment/config/s6/pimcore-automations/pimcore-automations /etc/s6-overlay/s6-rc.d/user/contents.d
