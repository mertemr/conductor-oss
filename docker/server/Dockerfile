# syntax=docker/dockerfile:1.7.0-labs

#
# conductor:server - Build Conductor Server
#
# ===========================================================================================================
# 0. Builder stage 1
# ===========================================================================================================
FROM openjdk:17-bullseye AS builder

LABEL maintainer="Orkes OSS <oss@orkes.io>"

# Copy the project directly onto the image
WORKDIR /builder

COPY --exclude=docker/ --exclude=ui/ . .

# Build the server on run, using Docker build cache for Gradle dependencies
RUN --mount=type=cache,target=/root/.gradle/caches \
    --mount=type=cache,target=/root/.gradle/wrapper \
    ./gradlew build -x test

#
# conductor:server - Build Conductor UI
#
# ===========================================================================================================
# 1. Builder stage 2
# ===========================================================================================================
FROM alpine:3.20 AS ui-builder

WORKDIR /ui

ARG YARN_OPTS=""

LABEL maintainer="Orkes OSS <oss@orkes.io>"

# Install dependencies
RUN apk add --no-cache \
    nodejs npm yarn && \
    rm -rf /var/cache/apk/*

# Include monaco sources into bundle (instead of using CDN)
ENV REACT_APP_MONACO_EDITOR_USING_CDN=false \
    REACT_APP_ENABLE_ERRORS_INSPECTOR=true

RUN --mount=type=cache,target=/ui/node_modules \
    --mount=type=cache,target=/root/.yarn \
    --mount=type=bind,source=./ui/package.json,target=package.json \
    --mount=type=bind,source=./ui/yarn.lock,target=yarn.lock \
    --mount=type=bind,source=./ui/public,target=public \
    yarn ${YARN_OPTS:-} install && cp -r node_modules/monaco-editor public/

ENV YARN_CACHE_FOLDER=/root/.yarn

COPY ./ui .

RUN --mount=type=cache,target=/root/.yarn \
    --mount=type=cache,target=/ui/node_modules \
    yarn ${YARN_OPTS:-} --frozen-lockfile build


# ===========================================================================================================
# 2. Bin stage
# ===========================================================================================================
FROM alpine:3.22 AS server

LABEL maintainer="Orkes OSS <oss@orkes.io>"

RUN apk add --no-cache \
    curl ca-certificates nginx && \
    update-ca-certificates && \
    rm -rf /var/cache/apk/*

WORKDIR /app

# Make app folders
RUN mkdir -p ./config ./logs ./libs

# Copy the compiled output to new image
COPY docker/server/bin .
COPY docker/server/config ./config

COPY --from=builder /builder/server/build/libs/*boot*.jar ./libs/conductor-server.jar

COPY docker/server/nginx/nginx.conf /etc/nginx/http.d/default.conf

# Copy compiled UI assets to nginx www directory
WORKDIR /usr/share/nginx/html
RUN rm -rf ./*

COPY --from=ui-builder /ui/build .

# Copy the files for the server into the app folders
RUN chmod +x /app/startup.sh

HEALTHCHECK --interval=60s --timeout=30s --retries=10 \
    CMD curl --fail http://localhost:8080/health || exit 1

CMD [ "/app/startup.sh" ]
ENTRYPOINT [ "/bin/sh"]
