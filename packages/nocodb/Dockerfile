###
# Minimal Dockerfile for testing nocodb forks (the OSS repo doesn't ship one).
# Builds with the dev rspack pipeline — fine for a test instance, not tuned
# for prod image size. Run from repo root: `docker build -f packages/nocodb/Dockerfile .`
###

FROM node:22-bookworm-slim AS base

# Tools needed for some native deps + healthcheck.
RUN apt-get update && apt-get install -y --no-install-recommends \
    python3 \
    make \
    g++ \
    git \
    ca-certificates \
    curl \
  && rm -rf /var/lib/apt/lists/*

# Pin pnpm to the version the repo expects.
RUN npm install -g pnpm@10.33.2

# Tells pnpm we're in non-interactive mode — required for nested
# pnpm install in noco-integrations (it tries to confirm modules dir
# removal otherwise and aborts: ERR_PNPM_ABORTED_REMOVE_MODULES_DIR_NO_TTY).
ENV CI=true

WORKDIR /app

# Copy workspace manifests first so Docker layer-caches the install step.
COPY pnpm-workspace.yaml pnpm-lock.yaml package.json ./
COPY packages/nocodb/package.json packages/nocodb/
COPY packages/nocodb-sdk/package.json packages/nocodb-sdk/
COPY packages/nocodb-sdk-v2/package.json packages/nocodb-sdk-v2/
COPY packages/nc-gui/package.json packages/nc-gui/
COPY packages/nc-secret-mgr/package.json packages/nc-secret-mgr/

# Install the OSS workspace deps (skip the nc-gui postinstall scripts that
# don't matter for the API server; we only need the nocodb package + sdk).
RUN pnpm install --frozen-lockfile --ignore-scripts

# Bring in the source.
COPY packages/nocodb-sdk packages/nocodb-sdk
COPY packages/nocodb-sdk-v2 packages/nocodb-sdk-v2
COPY packages/nocodb packages/nocodb
COPY packages/noco-integrations packages/noco-integrations
# SDK build's `generate:sdk` step reads templates from the workspace's
# scripts/ dir (referenced as `../../scripts/sdk/templates` from
# packages/nocodb-sdk). Without these the generator falls back to its
# default template, producing an Api.ts incompatible with the source.
COPY scripts scripts

# Build the SDK + integrations core (both required at nocodb runtime).
RUN cd packages/nocodb-sdk && pnpm build
# noco-integrations has its OWN pnpm workspace (isolated from the outer
# one) that core/ depends on. core/src/utils/axios.ts imports `axios`
# but core/package.json doesn't declare it — locally tsc resolves it
# via the symlinked nocodb-sdk's transitive deps. In Docker that
# resolution fails, so we --shamefully-hoist to flatten node_modules
# and make axios visible at the workspace root.
RUN cd packages/noco-integrations \
  && pnpm install --frozen-lockfile --shamefully-hoist \
  && cd core && pnpm build

# Bundle nocodb itself. The dev rspack config wires `RunScriptWebpackPlugin`
# which runs the bundle after build — we exploit that as the runtime entry.
WORKDIR /app/packages/nocodb

ENV NODE_ENV=development \
    NC_DISABLE_TELE=true \
    PORT=8080 \
    NC_TOOL_DIR=/usr/app/data

EXPOSE 8080

# Persisted SQLite + uploads.
VOLUME ["/usr/app/data"]
RUN mkdir -p /usr/app/data

# Start the server (rspack bundles, then RunScriptWebpackPlugin invokes the
# bundle). Works as a long-running server because rspack stays in watch mode,
# but with no file changes inside the container it just sits idle after the
# initial compile.
CMD ["pnpm", "start"]
