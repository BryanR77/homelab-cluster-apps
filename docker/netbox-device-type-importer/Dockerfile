FROM python:3.11-slim

# Install git at build time (no capability restrictions during image build)
RUN apt-get update && apt-get install -y --no-install-recommends \
        git ca-certificates \
    && rm -rf /var/lib/apt/lists/*

# Install Python dependencies (pynetbox>=7.3.4 required for NetBox 4.x)
RUN pip install --no-cache-dir "pynetbox>=7.3.4" requests pyyaml

# Clone the official netbox-community importer (develop branch, NetBox 4.x compatible)
RUN git clone --depth 1 -b develop \
    https://github.com/netbox-community/Device-Type-Library-Import.git /importer

# Run as non-root
RUN useradd -u 1000 -m appuser
USER appuser

WORKDIR /tmp

CMD ["python3", "/importer/nb-dt-import.py"]
