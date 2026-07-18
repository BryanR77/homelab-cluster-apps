# NetBox image with the netbox-proxbox plugin (https://github.com/emersonfelipesp/netbox-proxbox)
# pip-installed into the official image's venv.
#
# NETBOX_VERSION must track the appVersion of the netbox chart version pinned in
# homelab-cluster-apps/apps/netbox.yaml (currently chart 8.3.37 -> NetBox v4.6.5).
# Bump it whenever that chart version changes.
ARG NETBOX_VERSION=v4.6.5
FROM ghcr.io/netbox-community/netbox:${NETBOX_VERSION}

# The base image's venv is created with `uv venv` (no --seed), so it has no pip
# binary at all. Use uv itself (same tool the base image is built with) to
# install into that venv instead.
COPY --from=ghcr.io/astral-sh/uv:0.11 /uv /usr/local/bin/uv
RUN uv pip install --python /opt/netbox/venv/bin/python --no-cache netbox-proxbox
