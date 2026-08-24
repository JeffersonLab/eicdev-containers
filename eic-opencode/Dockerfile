# EIC + opencode agent layer (adds Node.js + opencode on top of eic-full)
#
# Turns the full EIC software stack into an environment where the opencode
# agent can build, run, and debug EIC software either interactively, in a
# semi-automatic mode (auto-accept edits, prompt for shell commands), or in a
# fully autonomous / headless mode (--auto, no prompts).
#
# opencode is provider-agnostic: it picks a provider from whichever API key is
# present in the environment (ANTHROPIC_API_KEY, OPENAI_API_KEY,
# GEMINI_API_KEY, OPENROUTER_API_KEY, ...), or from an auth.json created by
# `opencode providers login`.  No default model is baked in; pass
# `-m provider/model` to pin one.
#
# Build:
#   docker build -t eicdev/eic-opencode:latest .
#   docker buildx build --tag eicdev/eic-opencode:latest --push .
#
# Run — interactive agent (bring your own key):
#   docker run --rm -it --init \
#     -e ANTHROPIC_API_KEY=sk-ant-... \
#     -v "$PWD":/work -w /work \
#     eicdev/eic-opencode:latest opencode
#
# Run — semi-automatic (default settings: auto-accept edits, ask before running
#        shell commands):
#   docker run --rm -it --init -e ANTHROPIC_API_KEY=sk-ant-... \
#     -v "$PWD":/work -w /work eicdev/eic-opencode:latest \
#     opencode run "fix the failing EICrecon test"
#
# Run — fully autonomous / headless (no prompts at all):
#   docker run --rm --init -e ANTHROPIC_API_KEY=sk-ant-... \
#     -v "$PWD":/work -w /work eicdev/eic-opencode:latest \
#     opencode run --auto "build epic and report any errors"
#
# Run — autoresearch optimization loop:
#   docker run --rm -it --init \
#     --user $(id -u):$(id -g) \
#     -e HOME=/myhome \
#     -e ANTHROPIC_API_KEY=$ANTHROPIC_API_KEY \
#     -e UV_CACHE_DIR=/data/autoresearch/uv-cache \
#     -v ~/.opencode-docker:/myhome \
#     -v /path/to/data:/data \
#     -v /path/to/autoresearch-for-you:/work \
#     -w /work \
#     eicdev/eic-opencode:latest \
#     opencode run --auto "$(cat /work/optimize.md)"
#
#   (opencode creates ~/.config/opencode, ~/.local/share/opencode,
#    ~/.local/state/opencode and ~/.cache/opencode at startup, so for non-root
#    runs the WHOLE home must be writable — mount over /myhome, not over a
#    single subdirectory.)

# uv build stage — declared here so VERSION_UV is in global scope.  Docker
# forbids variable expansion in `COPY --from=...`, so we pull uv as a named
# stage (which *does* allow an ARG in the FROM) and copy from that stage below.
# Pin with --build-arg VERSION_UV=0.5.0 (an image tag) for reproducible builds.
ARG VERSION_UV=latest
FROM ghcr.io/astral-sh/uv:${VERSION_UV} AS uv

FROM eicdev/eic-full:latest

WORKDIR ${APP_ROOT}

# ---------------------------------------------------------------------------#
#  Node.js (opencode runtime dependency)                                     #
#                                                                            #
#  The opencode npm package needs Node.js 20+.  Ubuntu 24.04 ships an older  #
#  node in its repos, so we install the current LTS (22.x) from NodeSource.  #
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
#  opencode                                                                   #
#                                                                            #
#  Installed globally via npm.  Pin a specific version with                  #
#  --build-arg VERSION_OPENCODE=1.18.21 for reproducible builds.             #
# ---------------------------------------------------------------------------#
ARG VERSION_OPENCODE=latest

RUN set -ex && \
    npm install -g "opencode-ai@${VERSION_OPENCODE}" && \
    npm cache clean --force && \
    opencode --version

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
#  opencode.json ships a sensible "semi-automatic" default: reads, searches   #
#  and edits are auto-approved, but shell commands still prompt.  Autonomous  #
#  runs override this per-invocation with `opencode run --auto`.              #
#                                                                            #
#  Two copies are baked.  /root/.config/opencode/opencode.json is the XDG      #
#  default picked up by root runs.  /etc/opencode/opencode.json is the one    #
#  OPENCODE_CONFIG points at: /root is mode 700, so a `--user $(id -u)` run   #
#  cannot read the copy under /root, and would otherwise silently lose the    #
#  baked defaults.  Override OPENCODE_CONFIG to ship a different config.      #
#                                                                            #
#  OPENCODE_DISABLE_AUTOUPDATE pins the baked-in CLI version for              #
#  reproducibility (opencode.json sets the same thing via "autoupdate").      #
# ---------------------------------------------------------------------------#
COPY opencode.json /root/.config/opencode/opencode.json
COPY opencode.json /etc/opencode/opencode.json

ENV OPENCODE_CONFIG=/etc/opencode/opencode.json \
    OPENCODE_DISABLE_AUTOUPDATE=1

# Non-interactive git identity so autonomous commits do not fail.
RUN git config --global user.name  "EIC OpenCode Agent" && \
    git config --global user.email "eic-opencode@localhost" && \
    git config --global init.defaultBranch main && \
    git config --global --add safe.directory '*'

SHELL ["/bin/bash", "-c"]
