# syntax=docker/dockerfile:1.7

ARG PHP_IMAGE=ghcr.io/digital-entropy/dokar-php/octane:8.5

FROM ${PHP_IMAGE} AS php-dependencies
WORKDIR /var/www
ENV COMPOSER_CACHE_DIR=/tmp/composer-cache
COPY composer.json composer.lock ./
RUN --mount=type=cache,target=/tmp/composer-cache,sharing=locked \
    composer install --no-dev --no-interaction --no-scripts --prefer-dist
COPY . .
RUN composer dump-autoload --no-dev --classmap-authoritative --no-interaction

FROM node:24-alpine AS frontend
WORKDIR /var/www
ENV PNPM_HOME=/pnpm PATH=/pnpm:$PATH
RUN corepack enable
COPY package.json pnpm-lock.yaml pnpm-workspace.yaml* ./
RUN --mount=type=cache,target=/pnpm/store,sharing=locked \
    pnpm install --frozen-lockfile --store-dir=/pnpm/store
COPY . .
ARG VITE_PUBLIC_URL
ENV VITE_PUBLIC_URL=${VITE_PUBLIC_URL}
RUN pnpm run build

FROM ${PHP_IMAGE} AS production
ARG APP_UID=1000
ARG APP_GID=1000
ARG APP_USER=app
WORKDIR /var/www
COPY --from=php-dependencies --chown=${APP_UID}:${APP_GID} /var/www /var/www
COPY --from=frontend --chown=${APP_UID}:${APP_GID} /var/www/public/build /var/www/public/build
RUN mkdir -p storage/framework/cache storage/framework/sessions storage/framework/views bootstrap/cache \
    && chown -R ${APP_UID}:${APP_GID} storage bootstrap/cache
USER ${APP_USER}
EXPOSE 80
HEALTHCHECK --interval=10s --timeout=3s --retries=12 CMD curl -fsS http://127.0.0.1/healthz || exit 1
CMD ["/runner.sh"]
