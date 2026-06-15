### Checkout Service written in Go

# ─── STAGE 1: BUILDER ──────────────────────────────────────────────────────────

ARG BUILDPLATFORM=linux/amd64

FROM --platform=$BUILDPLATFORM golang:1.26.2-alpine@sha256:f85330846cde1e57ca9ec309382da3b8e6ae3ab943d2739500e08c86393a21b1 AS builder

# TARGETOS and TARGETARCH are Docker BuildKit built-ins — they automatically get
# set to the TARGET platform (where the image will RUN), not where you're building.
# We declare them as ARG so we can use them in the RUN command below.
# Defaults: linux/amd64 (standard server/k8s environment)
ARG TARGETOS=linux
ARG TARGETARCH=amd64

WORKDIR /src

# Same layer caching trick as Node's package.json pattern —
# copy dependency manifest files first, then download.
# go.mod → defines which packages are needed (like package.json)
# go.sum → contains checksums to verify downloaded packages weren't tampered with (like package-lock.json)
# As long as these don't change, `go mod download` layer is cached.
COPY go.mod go.sum ./
RUN go mod download

COPY . .

# Skaffold is a local K8s dev tool — it can pass special compiler flags for
# debugging (e.g., disabling optimizations so you can attach a debugger).
# In normal builds this will just be empty, so it has no effect.
ARG SKAFFOLD_GO_GCFLAGS

# ─── THE MAIN BUILD COMMAND ────────────────────────────────────────────────────
# Breaking it down piece by piece:
#
# GOOS=${TARGETOS} GOARCH=${TARGETARCH}
#   → Cross-compilation env vars. Tell the Go compiler what OS and CPU
#     architecture to build FOR.
#   → This is Go's superpower — you can build a Linux/amd64 binary right
#     from your Mac without any extra tools. No equivalent in Node.
#   → e.g., GOOS=linux GOARCH=amd64 on an M2 Mac → produces a binary that
#     runs on your amd64 Linux k8s node. This is why Go is loved in DevOps.
#
# CGO_ENABLED=0
#   → CGO allows Go code to call C libraries. Disabling it means the compiler
#     produces a FULLY STATIC binary — zero external dependencies.
#   → WHY this matters: with CGO_ENABLED=1, your binary links against C libs
#     on the host (like glibc). If those libs aren't in the final image, it crashes.
#   → With CGO_ENABLED=0 → the binary is self-contained. You can drop it into
#     a completely empty image (distroless/scratch) and it just runs.
#
# go build
#   → Compiles the entire Go application into a single binary.
#   → Unlike Node, there's no runtime needed in the final image — the binary
#     IS the application. This is the key difference from interpreted languages.
#
# -ldflags="-s -w"
#   → Linker flags that strip debug info from the binary:
#   → -s → strips the symbol table (function names, types used by debuggers)
#   → -w → strips DWARF debug info (line numbers, variable info)
#   → Result: binary is ~30% smaller. You don't want debug symbols in prod —
#     they bloat the binary and expose internal implementation details.
#
# -gcflags="${SKAFFOLD_GO_GCFLAGS}"
#   → Compiler flags, passed through from Skaffold for debug builds.
#   → In prod/normal builds this is empty — no effect.
#
# -o /checkoutservice
#   → Output path for the compiled binary. Placing it at root `/` makes it
#     easy to COPY across stages without worrying about nested paths.
#
# . (dot at the end)
#   → "Build the package in the current directory" — tells go build where
#     the main() function lives.
RUN GOOS=${TARGETOS} GOARCH=${TARGETARCH} CGO_ENABLED=0 \
    go build \
    -ldflags="-s -w" \
    -gcflags="${SKAFFOLD_GO_GCFLAGS}" \
    -o /checkoutservice .


# ─── STAGE 2: FINAL / RUNTIME IMAGE ───────────────────────────────────────────
# distroless/static → Google's image that contains LITERALLY NOTHING except
# the bare minimum to run a static binary (ca-certificates, tzdata).
# No shell (/bin/sh), no package manager, no utilities, nothing.
#
# Compare the chain of minimalism:
# ubuntu (~80MB) → alpine (~7MB) → distroless/static (~2MB) → scratch (0MB)
#
# WHY distroless over scratch?
# scratch is truly empty — no ca-certificates (HTTPS calls would fail),
# no timezone data. distroless/static includes those two essentials.
# WHY distroless over alpine?
# No shell = no shell injection attacks. If someone compromises the container,
# they can't `exec` into it and run commands. Smaller attack surface.
FROM gcr.io/distroless/static

WORKDIR /src

# Copy ONLY the compiled binary from stage 1.
# This is the entire application — one file.
# No node_modules, no source code, no build tools, nothing else.
COPY --from=builder /checkoutservice /src/checkoutservice

# Controls how Go prints stack traces on crashes/panics.
# "single" → only prints the stack trace of the goroutine that crashed.
# Default Go behavior dumps ALL goroutines — which in a microservice with
# hundreds of concurrent goroutines creates massive, unreadable crash logs.
ENV GOTRACEBACK=single

EXPOSE 5050

# No shell in distroless, so exec form is the only option anyway.
# The binary IS the process — no interpreter, no runtime, nothing in between.
ENTRYPOINT ["/src/checkoutservice"]