# Builds the static Homelab Docs site from markdown living in two repos:
#   - infra-k8s (this repo):  docs/
#   - infra (separate repo):  docs/, runbooks/
#
# Cloning the infra repo needs a token with read access, passed as a
# BuildKit secret (never baked into image layers/history) -- see
# .github/workflows/docs-site.yml, secret name INFRA_REPO_PAT.

FROM python:3.12-slim AS build
RUN apt-get update && apt-get install -y --no-install-recommends git \
    && rm -rf /var/lib/apt/lists/*
RUN pip install --no-cache-dir mkdocs-material

WORKDIR /site
COPY docs-build/mkdocs.yml /site/mkdocs.yml

# This repo's own docs
COPY docs /site/docs

# infra repo's docs, cloned at build time (not vendored/committed here,
# so it can never drift out of date against the source repo)
RUN --mount=type=secret,id=infra_repo_pat \
    git clone --depth 1 \
      "https://x-access-token:$(cat /run/secrets/infra_repo_pat)@github.com/helloWorld44-89/infra.git" \
      /tmp/infra-src && \
    mkdir -p /site/docs/proxmox-ansible/network && \
    cp /tmp/infra-src/docs/README.md /site/docs/proxmox-ansible/README.md && \
    cp /tmp/infra-src/docs/network/corosync-redundancy.md /site/docs/proxmox-ansible/network/corosync-redundancy.md && \
    cp /tmp/infra-src/runbooks/README.md /site/docs/proxmox-ansible/runbooks-readme.md && \
    cp /tmp/infra-src/runbooks/CHANGELOG.md /site/docs/proxmox-ansible/CHANGELOG.md && \
    rm -rf /tmp/infra-src

# Not --strict: monitoring-setup.md has pre-existing links to docs that
# were planned but never written (flux/README.md, ingress/README.md,
# monitoring.md) and to a non-markdown config file. Real content gaps
# in the source docs, not a build problem -- don't block the pipeline
# on them, but they're still worth fixing in those source files.
RUN mkdocs build -d /site/built

FROM nginx:alpine
COPY --from=build /site/built /usr/share/nginx/html
EXPOSE 80
