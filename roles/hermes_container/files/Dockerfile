# Dockerfile for Hermes Agent container
FROM ubuntu:22.04 AS base
ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        ca-certificates curl git python3 python3-venv \
        libssl-dev libffi-dev build-essential \
        wget gnupg lsb-release jq \
        rocm-opencl rocm-dev rocm-utils && \
    rm -rf /var/lib/apt/lists/*

# Install Ollama (optional, can be removed if not needed)
ARG OLLAMA_VERSION=0.3.13
RUN curl -fsSL https://ollama.com/install.sh | sh

# Build llama.cpp
ARG LLAMA_CPP_TAG=v0.2.35
RUN git clone --depth 1 --branch ${LLAMA_CPP_TAG} https://github.com/ggerganov/llama.cpp /opt/llama.cpp && \
    cd /opt/llama.cpp && make -j$(nproc) && \
    cp llama-server /usr/local/bin/ && cp llama-cli /usr/local/bin/

# Install Hermes Agent via official installer
ENV HERMES_HOME=/hermes
RUN mkdir -p ${HERMES_HOME}
WORKDIR ${HERMES_HOME}
RUN curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash
ENV PATH="${HERMES_HOME}/.hermes/bin:${PATH}"

# Create non‑root user (will be matched to host UID/GID at runtime)
ARG UID=1000
ARG GID=1000
RUN groupadd -g ${GID} hermes && \
    useradd -m -u ${UID} -g ${GID} -s /bin/bash hermes && \
    chown -R hermes:hermes ${HERMES_HOME}
USER hermes

CMD ["/bin/bash"]
