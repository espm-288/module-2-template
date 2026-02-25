FROM ghcr.io/rocker-org/ml

# Install Node.js (provides npx, required for Context7 MCP server)
RUN apt-get update && apt-get install -y --no-install-recommends nodejs npm \
    && npm install -g npm@latest \
    && rm -rf /var/lib/apt/lists/*

# Additional R packages for cloud-native geospatial workflows
RUN install2.r --error --skipinstalled \
    gdalcubes \
    rstac \
    duckdbfs \
    mapgl \
    && rm -rf /tmp/downloaded_packages

