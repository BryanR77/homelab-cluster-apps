FROM python:3.13-slim

# Install git at build time (no capability restrictions during image build)
RUN apt-get update && apt-get install -y --no-install-recommends \
        git ca-certificates \
    && rm -rf /var/lib/apt/lists/*

# Clone the official netbox-community importer (develop branch, NetBox 4.x compatible)
RUN git clone --depth 1 -b develop \
    https://github.com/netbox-community/Device-Type-Library-Import.git /importer

# Install dependencies from the importer's own requirements.txt, pinning pynetbox
# for NetBox 4.x compatibility (>=7.3.4 fixes token passing against NetBox 4.x API)
RUN pip install --no-cache-dir -r /importer/requirements.txt "pynetbox>=7.3.4"

# Run as non-root
RUN useradd -u 1000 -m appuser
USER appuser

WORKDIR /tmp

CMD ["python3", "/importer/nb-dt-import.py"]
