ARG OPENCLAW_VERSION=v2026.5.2
FROM ghcr.io/openclaw/openclaw:${OPENCLAW_VERSION}

# Bake messaging-channel plugins into the image so they survive container
# recreation. Plugins are written under /home/node/.openclaw/plugins, which is
# the persistent named volume mount point in docker-compose.yml — Docker
# pre-populates a fresh named volume from the image on first creation, so the
# plugin is present on initial install and persists across restarts.
#
# To add more plugins (e.g. Slack, Discord), append more `openclaw plugins
# install` lines below and bump the image tag in docker-compose.yml so the
# build is re-triggered.
RUN openclaw plugins install @openclaw/matrix
