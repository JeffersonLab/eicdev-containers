# EIC + Gemini CLI agent layer (adds Node.js + Gemini CLI on top of eic-full)
#
# Turns the full EIC software stack into an environment where the Gemini CLI
# agent can build, run, and debug EIC software either interactively, in a
# semi-automatic mode (auto-accept edits, prompt for risky commands), or in a
# fully autonomous / headless mode (--yolo, no prompts).
#
# Build:
#   docker build -t eicdev/eic-gemini:latest .
#   docker buildx build --tag eicdev/eic-gemini:latest --push .
#
# Run — interactive agent (bring your own key):
#   docker run --rm -it --init \
#     -e GEMINI_API_KEY=AIzaSy... \
#     -v "$PWD":/work -w /work \
#     eicdev/eic-gemini:latest gemini
#
# Run — semi-automatic (default settings: auto-accept edits, ask before running
#        shell commands):
#   docker run --rm -it --init -e GEMINI_API_KEY=AIzaSy... \
#     -v "$PWD":/work -w /work eicdev/eic-gemini:latest \
#     gemini -p "fix the failing EICrecon test"
#
# Run — fully autonomous / headless (no prompts at all):
#   docker run --rm --init -e GEMINI_API_KEY=AIzaSy... \
#     -v "$PWD":/work -w /work eicdev/eic-gemini:latest \
#     gemini --yolo -p "build epic and report any errors"
#
# Run — autoresearch optimization loop:
#   docker run --rm -it --init \
#     --user $(id -u):$(id -g) \
#     -e HOME=/myhome \
#     -e GEMINI_CONFIG_DIR=/myhome/.gemini \
#     -e GEMINI_API_KEY=$GEMINI_API_KEY \
#     -e UV_CACHE_DIR=/data/autoresearch/uv-cache \
#     -v ~/.gemini-docker:/myhome/.gemini \
#     -v /path/to/data:/data \
#     -v /path/to/autoresearch-for-you:/work \
#     -w /work \
#     eicdev/eic-gemini:latest \
#     gemini --yolo -p "$(cat /work/optimize.md)"

# uv build stage — declared here so VERSION_UV is in global scope.  Docker
# forbids variable expansion in `COPY --from=...`, so we pull uv as a named
# stage (which *does* allow an ARG in the FROM) and copy from that stage below.
# Pin with --build-arg VERSION_UV=0.5.0 (an image tag) for reproducible builds.
ARG VERSION_UV=latest
FROM ghcr.io/astral-sh/uv:${VERSION_UV} AS uv

FROM eicdev/eic-full:latest

WORKDIR ${APP_ROOT}

# ---------------------------------------------------------------------------#
#  Node.js (Gemini CLI runtime dependency)                                   #
#                                                                            #
#  Gemini CLI needs Node.js 20+.  Ubuntu 24.04 ships an older node in its    #
#  repos, so we install the current LTS (22.x) from NodeSource.              #
# ---------------------------------------------------------------------------#
ARG NODE_MAJOR=22

RUN set -ex && \
    apt-get update && \
    apt-get install -y ca-certificates gnupg && \
    mkdir -p /etc/apt/keyrings && \
    curl -fsSL https://deb.nodesource.com/gpgkey/nodesource-repo.gpg.key \
        | gpg --dearmor -o /etc/apt/keyrings/nodesource.gpg && \
    echo "deb [signed-by=/etc/apt/keyrings/nodesource.gpg] https://deb.nodesource.com/node_${NODE_MAJOR}.x nodistro main" \
        > /etc/apt/sources.list.d/nodesource.list && \
    apt-get update && \
    apt-get install -y nodejs && \
    apt-get clean && apt-get autoremove -y && rm -rf /var/lib/apt/lists/*

# ---------------------------------------------------------------------------#
#  Gemini CLI                                                                #
#                                                                            #
#  Installed globally via npm.  Pin a specific version with                  #
#  --build-arg VERSION_GEMINI_CLI=0.56.0 for reproducible builds.            #
# ---------------------------------------------------------------------------#
ARG VERSION_GEMINI_CLI=latest

RUN set -ex && \
    npm install -g "@google/gemini-cli@${VERSION_GEMINI_CLI}" && \
    npm cache clean --force && \
    gemini --version

# ---------------------------------------------------------------------------#
#  uv (Python package/venv manager used by agent workflows)                  #
#                                                                            #
#  Copied as static binaries from the `uv` stage defined at the top.         #
# ---------------------------------------------------------------------------#
COPY --from=uv /uv /uvx /usr/local/bin/

RUN uv --version

# ---------------------------------------------------------------------------#
#  Pre-install the project's Python dependencies into the uv cache            #
#                                                                            #
#  A dummy pyproject.toml (mirroring the target project) is synced at build   #
#  time.  This downloads every wheel — including CUDA-enabled torch — into    #
#  uv's cache (/root/.cache/uv), which persists in the image.  At run time    #
#  the agent's `uv sync` inside /work resolves entirely from cache with NO    #
#  network downloads, so it never waits.                                      #
#                                                                            #
#  --no-install-project: install only the dependencies, not the dummy root   #
#  package (there is no source to build).  The throwaway .venv is removed;    #
#  only the cache is what we keep.                                            #
#                                                                            #
#  NOTE: this adds several GB to the image (torch bundles the CUDA runtime).  #
#  Keep this file in sync with the real project's pyproject.toml so the       #
#  cached versions match what the agent resolves at run time.                 #
# ---------------------------------------------------------------------------#
COPY pyproject.toml /opt/uv-prime/pyproject.toml
RUN cd /opt/uv-prime && \
    uv sync --no-install-project && \
    rm -rf /opt/uv-prime/.venv

# ---------------------------------------------------------------------------#
#  Agent configuration                                                        #
#                                                                            #
#  settings.json ships a sensible "semi-automatic" default: edits are        #
#  auto-accepted, but shell commands still prompt.  Autonomous runs override #
#  this per-invocation with `gemini --yolo` (or `--approval-mode=yolo`).     #
# ---------------------------------------------------------------------------#
COPY settings.json /root/.gemini/settings.json
COPY settings.json /etc/gemini-cli/settings.json

ENV IS_SANDBOX=1 \
    GEMINI_CONFIG_DIR=/root/.gemini

# Non-interactive git identity so autonomous commits do not fail.
RUN git config --global user.name  "EIC Gemini Agent" && \
    git config --global user.email "eic-gemini@localhost" && \
    git config --global init.defaultBranch main && \
    git config --global --add safe.directory '*'

SHELL ["/bin/bash", "-c"]
