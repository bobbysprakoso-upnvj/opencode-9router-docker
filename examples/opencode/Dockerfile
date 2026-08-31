FROM ghcr.io/anomalyco/opencode:latest

USER root

RUN apk add --no-cache \
    git \
    nodejs \
    npm \
 && mkdir -p /home/opencode/.gsd \
 && chown 1000:1000 /home/opencode/.gsd

USER 1000:1000