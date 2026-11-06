# NetBox image with the netbox-proxbox (https://github.com/emersonfelipesp/netbox-proxbox)
# and netbox-diode (https://github.com/netboxlabs/diode-netbox-plugin) plugins
# pip-installed into the official image's venv.
#
# NETBOX_VERSION must track the appVersion of the netbox chart version pinned in
# homelab-cluster-apps/apps/netbox.yaml (currently chart 8.3.37 -> NetBox v4.6.5).
# Bump it whenever that chart version changes. netbox-diode-plugin >=1.12.0 is
# required for NetBox >=4.6.0; an unpinned install always picks up a compatible
# release.
ARG NETBOX_VERSION=v4.6.5
FROM ghcr.io/netbox-community/netbox:${NETBOX_VERSION}

# The base image's venv is created with `uv venv` (no --seed), so it has no pip
# binary at all. Use uv itself (same tool the base image is built with) to
# install into that venv instead.
COPY --from=ghcr.io/astral-sh/uv:0.11 /uv /usr/local/bin/uv
RUN uv pip install --python /opt/netbox/venv/bin/python --no-cache \
    netbox-proxbox \
    netboxlabs-diode-netbox-plugin

# The base image runs collectstatic at build time, before these plugins exist,
# so their JS/CSS never reached /opt/netbox/netbox/static and NetBox served
# 404s for them (proxbox's endpoint status badges spun forever and its
# connection-approval panel never loaded). Collect again with the plugins
# enabled, as netbox-docker documents for plugins with static files. The
# temporary plugins.py is restored afterwards; at runtime the chart's own
# config provides PLUGINS.
RUN cp /etc/netbox/config/plugins.py /tmp/plugins.py.orig \
 && printf 'PLUGINS = ["netbox_proxbox", "netbox_diode_plugin"]\n' > /etc/netbox/config/plugins.py \
 && DEBUG=true SECRET_KEY="build-time-only-collectstatic-dummy-key-000000000000" \
    /opt/netbox/venv/bin/python /opt/netbox/netbox/manage.py collectstatic --no-input \
 && mv /tmp/plugins.py.orig /etc/netbox/config/plugins.py \
 && test -f /opt/netbox/netbox/static/netbox_proxbox/js/connection-authority.js
