# NetBox image with the netbox-proxbox plugin (https://github.com/emersonfelipesp/netbox-proxbox)
# pip-installed into the official image's venv.
#
# NETBOX_VERSION must track the appVersion of the netbox chart version pinned in
# homelab-cluster-apps/apps/netbox.yaml (currently chart 8.3.37 -> NetBox v4.6.5).
# Bump it whenever that chart version changes.
ARG NETBOX_VERSION=v4.6.5
FROM ghcr.io/netbox-community/netbox:${NETBOX_VERSION}

RUN /opt/netbox/venv/bin/pip install --no-cache-dir netbox-proxbox
