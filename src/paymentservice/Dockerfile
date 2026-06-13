# ─── STAGE 0: BASE ─────────────────────────────────────────────────────────────
# Naming a base stage is a Python-specific pattern you'll see here.
# WHY: Both builder and final stage share the same base image.
# Instead of writing the full FROM line twice (and risking them drifting apart),
# you define it ONCE and both stages inherit from "base".
# This guarantees builder and runtime use the exact same Python version.
ARG BUILDPLATFORM=linux/amd64
FROM --platform=$BUILDPLATFORM python:3.14.4-alpine@sha256:dd4d2bd5b53d9b25a51da13addf2be586beebd5387e289e798e4083d94ca837a AS base


# ─── STAGE 1: BUILDER ──────────────────────────────────────────────────────────
FROM base AS builder

# PYTHONDONTWRITEBYTECODE=1 → stops Python from creating .pyc cache files
# (the __pycache__ folders you see in Python projects)
# Not needed in a container — they'd just bloat the image with no benefit.
#
# PYTHONUNBUFFERED=1 → forces Python's stdout/stderr to flush immediately.
# WHY this matters in containers: by default Python buffers output, so if your
# container crashes, the last few log lines might never get written.
# With this set, logs appear in real-time → critical for debugging in K8s.
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

# g++         → C++ compiler, needed to compile native extensions in some pip packages
# linux-headers → kernel header files, required by packages that interface with
#                 low-level system calls (e.g., grpcio compiles C extensions)
# Similar to Node's stage 1 — build tools needed to INSTALL packages, but not to RUN them.
# rm -rf /var/cache/apk/* → manually clean apk cache (Alpine's apk doesn't have --no-cache
# when using `apk update` separately, so we clean manually to keep layer small)
RUN apk update \
    && apk add --no-cache g++ linux-headers \
    && rm -rf /var/cache/apk/*

# Same layer caching pattern — copy dependency manifest first, install, then copy source.
# requirements.txt is Python's equivalent of package.json
COPY requirements.txt .
RUN pip install -r requirements.txt
# pip installs packages into /usr/local/lib/python3.14/site-packages/
# We'll cherry-pick just this folder in the final stage — leaving g++ and linux-headers behind.


# ─── STAGE 2: FINAL / RUNTIME IMAGE ───────────────────────────────────────────
# Back to "base" (the alpine python image) — NOT builder.
# So this stage has Python but none of the build tools (g++, linux-headers).
FROM base

ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

# This service can send traces/profiles to Google Cloud Profiler.
# Setting this to 1 enables that. Safe to ignore for local runs —
# it'll just fail silently if GCP credentials aren't present.
ENV ENABLE_PROFILER=1

# libstdc++ → the C++ standard library (RUNTIME version of g++)
# WHY: Stage 1 used g++ to COMPILE C++ extensions inside packages like grpcio.
# Those compiled extensions (.so files) still need libstdc++ to RUN —
# even though they don't need g++ itself anymore.
# This is the runtime vs build-time dependency distinction:
#   g++        → needed to compile  (build stage only)
#   libstdc++  → needed to execute  (runtime stage)
RUN apk update \
    && apk add --no-cache libstdc++ \
    && rm -rf /var/cache/apk/*

WORKDIR /email_server

# This is the KEY line — instead of copying node_modules (Node) or a binary (Go),
# Python copies its entire site-packages directory across from the builder.
# site-packages is where pip installs everything — it's Python's equivalent of node_modules.
# We get all the installed packages without bringing along the compiler that built them.
COPY --from=builder /usr/local/lib/python3.14/ /usr/local/lib/python3.14/

COPY . .

EXPOSE 8080

# Python IS the runtime here (like Node) — unlike Go where the binary runs standalone.
# The interpreter (python) is already in the base image, so we just point it at our script.
ENTRYPOINT [ "python", "email_server.py" ]