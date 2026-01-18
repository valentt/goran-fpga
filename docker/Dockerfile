# f32c Open-Source FPGA Build Environment
# Author: Hans Weber (FPGA Design Engineer)
# Date: 2026-01-18

FROM debian:bookworm-slim

LABEL maintainer="Hans Weber - FPGA Agent"
LABEL description="f32c synthesis environment with oss-cad-suite"

# Avoid interactive prompts
ENV DEBIAN_FRONTEND=noninteractive

# Install dependencies
RUN apt-get update && apt-get install -y \
    wget \
    curl \
    git \
    make \
    xz-utils \
    ca-certificates \
    libftdi1-2 \
    && rm -rf /var/lib/apt/lists/*

# Download and install oss-cad-suite
WORKDIR /opt
RUN wget -q https://github.com/YosysHQ/oss-cad-suite-build/releases/download/2024-12-17/oss-cad-suite-linux-x64-20241217.tgz \
    && tar -xzf oss-cad-suite-linux-x64-20241217.tgz \
    && rm oss-cad-suite-linux-x64-20241217.tgz

# Add oss-cad-suite to PATH
ENV PATH="/opt/oss-cad-suite/bin:${PATH}"

# Create working directory
WORKDIR /workspace

# Verify tools are working
RUN yosys --version && \
    ghdl --version && \
    nextpnr-ecp5 --version && \
    ecppack --version

# Default command
CMD ["/bin/bash"]
