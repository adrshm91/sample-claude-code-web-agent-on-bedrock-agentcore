# Use Node.js Trixie as base image
FROM node:trixie-slim

# Add uv to PATH
ENV PATH="/root/.local/bin:${PATH}"

RUN apt update && \
    apt install -y curl git git-lfs wget gh vim python3 python3-pip && \
    npm install -g @anthropic-ai/claude-code && \
    curl -LsSf https://astral.sh/uv/install.sh | sh && \
    echo "export VIRTUAL_ENV_DISABLE_PROMPT=1" >> /root/.bashrc && \
    echo "source /app/.venv/bin/activate" >> /root/.bashrc && \
    mkdir /workspace

COPY . /app/

# Set working directory
WORKDIR /app

# Install Python dependencies during build
RUN uv sync

# Expose port 8000
EXPOSE 8000

# Run the server on port 8000
CMD ["/app/serve"]
