### Currency service written in Node.js

# ─── STAGE 1: BUILDER ──────────────────────────────────────────────────────────
# ARG lets you pass a variable at build time (docker build --build-arg ...)
# Here it defaults to linux/amd64. This matters when you're building on an
# ARM Mac (M1/M2) but want the final image to run on amd64 servers/k8s nodes.
ARG BUILDPLATFORM=linux/amd64

# --platform=$BUILDPLATFORM → forces THIS stage to always run on amd64,
# regardless of what machine you're building on.
# WHY: some npm packages (like @google-cloud/profiler) have native binaries.
# If you compile them on ARM, they won't work on amd64. So we lock the
# BUILD stage to amd64 to avoid that cross-compilation mess.
#
# The SHA digest (@sha256:...) pins the EXACT image version.
# WHY: "node:20-alpine" is a moving tag — it can change tomorrow and break your build.
# The digest guarantees you always get the exact same base image. Critical for reproducibility.
#
# "AS builder" → names this stage. We'll reference it later in stage 2.
FROM --platform=$BUILDPLATFORM node:20.20.2-alpine@sha256:fb4cd12c85ee03686f6af5362a0b0d56d50c58a04632e6c0fb8363f609372293 AS builder

# Alpine is a minimal Linux distro — it doesn't come with python3, make, or g++.
# Some npm packages run "post-install scripts" that compile native C++ addons
# (e.g., @google-cloud/profiler). Those scripts need these build tools.
# --no-cache → don't cache the package index on disk (keeps image smaller)
RUN apk add --update --no-cache \
    python3 \
    make \
    g++

WORKDIR /usr/src/app

# Copy package.json AND package-lock.json first — BEFORE copying the rest of the source.
# WHY this order matters (Docker layer caching):
# Docker caches each instruction as a "layer". If package.json hasn't changed,
# Docker reuses the cached npm install layer and skips re-downloading all packages.
# If you copied ALL files first, any source code change would bust the npm cache.
COPY package*.json ./

# --only=production → skips devDependencies (test libs, build tools, etc.)
# WHY: smaller image, less attack surface in prod. Dev tools have no business
# being in a production container.
RUN npm install --only=production


# ─── STAGE 2: FINAL / RUNTIME IMAGE ───────────────────────────────────────────
# Fresh, clean alpine base — NOT the node image from stage 1.
# WHY a second stage (multi-stage build)?
# Stage 1 has python3, make, g++ installed — those are build tools, not needed at runtime.
# By starting fresh here and only copying what we need, the final image is MUCH smaller
# and has a smaller attack surface. This is standard production practice.
FROM alpine:3.23.4@sha256:5b10f432ef3da1b8d4c7eb6c487f2f5a8f096bc91145e68878dd4a5019afde11

# Install ONLY the nodejs runtime — no npm, no build tools, nothing else.
# We don't need npm at runtime; we just need node to run server.js.
RUN apk add --no-cache nodejs

WORKDIR /usr/src/app

# --from=builder → pulls files from Stage 1 (named "builder") instead of host machine.
# We only copy node_modules — the compiled, production-ready dependencies.
# The build tools (python3, make, g++) that were needed to install them are LEFT BEHIND
# in stage 1 and never make it into this final image.
COPY --from=builder /usr/src/app/node_modules ./node_modules

# Copy the actual application source code from the host (your local machine / CI context).
COPY . .

# Documents that this container listens on port 7000.
# EXPOSE is just metadata — it doesn't actually open the port.
# You still need -p 7000:7000 when running docker run, or a Service in K8s.
EXPOSE 7000

# ENTRYPOINT vs CMD:
# CMD → can be overridden at runtime (docker run myimage npm test)
# ENTRYPOINT → the container always runs this; args can be appended but not replaced easily.
# Using array form ["node", "server.js"] (exec form) is preferred over string form
# because it doesn't spin up a shell, so signals (SIGTERM, SIGKILL) go directly
# to the node process — important for graceful shutdown in K8s.
ENTRYPOINT [ "node", "server.js" ]