# NanoClaw Agent Container
# Runs GitHub Copilot SDK in isolated Linux VM with browser automation

FROM node:22-slim

# Install system dependencies for Chromium
RUN apt-get update && apt-get install -y \
    chromium \
    fonts-liberation \
    fonts-noto-color-emoji \
    libgbm1 \
    libnss3 \
    libatk-bridge2.0-0 \
    libgtk-3-0 \
    libx11-xcb1 \
    libxcomposite1 \
    libxdamage1 \
    libxrandr2 \
    libasound2 \
    libpangocairo-1.0-0 \
    libcups2 \
    libdrm2 \
    libxshmfence1 \
    curl \
    git \
    python3-pip \
    && rm -rf /var/lib/apt/lists/*

# Install tavily-cli (deep research CLI)
RUN pip3 install tavily-cli --break-system-packages

# Install infsh CLI (inference.sh — Tavily + Exa web search)
RUN curl -fsSL https://cli.inference.sh | sh

# Install kubectl
RUN curl -fsSL "https://dl.k8s.io/release/$(curl -fsSL https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl" \
    -o /usr/local/bin/kubectl \
    && chmod +x /usr/local/bin/kubectl

# Set Chromium path for agent-browser
ENV AGENT_BROWSER_EXECUTABLE_PATH=/usr/bin/chromium
ENV PLAYWRIGHT_CHROMIUM_EXECUTABLE_PATH=/usr/bin/chromium

# Add user local bin to PATH (for infsh, tvly, etc.)
ENV PATH="/root/.local/bin:/home/node/.local/bin:${PATH}"

# Install agent-browser globally
RUN npm install -g agent-browser

# Install GitHub CLI for Copilot SDK authentication
RUN curl -fsSL https://cli.github.com/packages/githubcli-archive-keyring.gpg | dd of=/usr/share/keyrings/githubcli-archive-keyring.gpg \
    && echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" | tee /etc/apt/sources.list.d/github-cli.list > /dev/null \
    && apt-get update && apt-get install -y gh && rm -rf /var/lib/apt/lists/*

# Create app directory
WORKDIR /app

# Copy package files first for better caching
COPY agent-runner/package*.json ./

# Install dependencies
RUN npm install

# Patch vscode-jsonrpc: add exports map so ESM can resolve 'vscode-jsonrpc/node'
# (The package ships node.js but has no exports field — Node 22 ESM strict mode requires it)
RUN node -e " \
  const fs = require('fs'); \
  const p = JSON.parse(fs.readFileSync('node_modules/vscode-jsonrpc/package.json', 'utf8')); \
  p.exports = { '.': './lib/node/main.js', './node': './node.js', './node.js': './node.js', './browser': './browser.js', './browser.js': './browser.js' }; \
  fs.writeFileSync('node_modules/vscode-jsonrpc/package.json', JSON.stringify(p, null, 2)); \
  console.log('Patched vscode-jsonrpc exports'); \
"

# Copy source code
COPY agent-runner/ ./

# Build TypeScript
RUN npm run build

# Create workspace directories
RUN mkdir -p /workspace/group /workspace/global /workspace/extra /workspace/ipc/messages /workspace/ipc/tasks /workspace/ipc/input

# Create entrypoint script
# Secrets are passed via stdin JSON — piped directly to Node (never written to disk)
# Follow-up messages arrive via IPC files in /workspace/ipc/input/
RUN printf '#!/bin/bash\nset -e\ncd /app && npx tsc --outDir /tmp/dist 2>&1 >&2\nln -s /app/node_modules /tmp/dist/node_modules\nchmod -R a-w /tmp/dist\nexec node /tmp/dist/index.js\n' > /app/entrypoint.sh && chmod +x /app/entrypoint.sh

# Set ownership to node user (non-root) for writable directories
RUN chown -R node:node /workspace && chmod 777 /home/node

# Switch to non-root user for security
USER node

# Set working directory to group workspace
WORKDIR /workspace/group

# Entry point reads JSON from stdin, outputs JSON to stdout
ENTRYPOINT ["/app/entrypoint.sh"]
