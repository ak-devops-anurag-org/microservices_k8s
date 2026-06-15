### Email service written in C#


# ─── STAGE 1: BUILDER ──────────────────────────────────────────────────────────
ARG BUILDPLATFORM=linux/amd64

# Microsoft's official .NET SDK image — has everything needed to BUILD .NET apps:
# the compiler (Roslyn), MSBuild, NuGet package manager, runtime, etc.
# "noble" = Ubuntu 24.04 LTS base (Noble Numbat). Microsoft uses Ubuntu for .NET images.
# This is heavier than alpine-based images but .NET tooling is built around Ubuntu/Debian.
FROM --platform=$BUILDPLATFORM mcr.microsoft.com/dotnet/sdk:10.0.100-noble@sha256:c7445f141c04f1a6b454181bd098dcfa606c61ba0bd213d0a702489e5bd4cd71 AS builder

ARG TARGETARCH=amd64
WORKDIR /app

# .csproj is C#'s project manifest — defines dependencies, target framework, build config.
# Equivalent of package.json (Node) / go.mod (Go) / requirements.txt (Python).
# Same layer caching pattern: copy manifest → restore deps → copy source → build.
COPY cartservice.csproj .

# "restore" = download all NuGet packages (C#'s equivalent of npm install / go mod download / pip install)
# -a $TARGETARCH → tells dotnet which CPU arch to restore native packages for.
# Some NuGet packages ship platform-specific native binaries (like grpcio in Python),
# so the target arch matters at restore time too.
RUN dotnet restore cartservice.csproj \
    -a $TARGETARCH

COPY . .

# "publish" = compile + bundle everything into a deployable output folder.
# In .NET, "build" just compiles. "publish" is what you actually deploy.
# Breaking down the flags — this is where C# gets interesting:
#
# -p:PublishSingleFile=true
#   → Bundles the entire app + all its DLLs into ONE executable file.
#   → Similar to Go's single binary output. Instead of a folder of 50 .dll files,
#     you get one file. Cleaner to copy across stages.
#
# -a $TARGETARCH
#   → Target CPU architecture. Same cross-compilation idea as Go's GOARCH.
#
# --self-contained true
#   → Bundles the .NET RUNTIME itself into the output binary.
#   → WHY: means the final image doesn't need the full .NET runtime installed —
#     only the native OS dependencies (libc, libssl etc.) which are tiny.
#   → Without this: final image must have dotnet runtime installed (~80MB extra).
#   → With this: runtime is baked in, final image can be minimal. Same idea as
#     CGO_ENABLED=0 in Go — making the output self-sufficient.
#
# -p:PublishTrimmed=true + -p:TrimMode=full
#   → Tree-shaking for .NET. The .NET runtime is huge — but your app only uses
#     a fraction of it. Trimming analyzes which parts of the runtime your code
#     actually calls and strips out everything else.
#   → TrimMode=full is the most aggressive setting — removes even more unused code.
#   → Can cut binary size by 60-70%. The tradeoff: reflection-heavy code can break
#     (trimmer can't always detect dynamically referenced types). Fine for this service.
#
# -c release
#   → Build in Release configuration (vs Debug).
#   → Enables compiler optimizations, removes debug assertions, no debug symbols.
#   → Equivalent to Go's -ldflags="-s -w" in effect (smaller, faster output).
#
# -o /cartservice
#   → Output directory for the published binary.
RUN dotnet publish cartservice.csproj \
    -p:PublishSingleFile=true \
    -a $TARGETARCH \
    --self-contained true \
    -p:PublishTrimmed=true \
    -p:TrimMode=full \
    -c release \
    -o /cartservice


# ─── STAGE 2: FINAL / RUNTIME IMAGE ───────────────────────────────────────────
# "runtime-deps" — NOT the full dotnet runtime image.
# There are 3 Microsoft .NET images to know:
#   sdk             → full build toolchain (~800MB) — Stage 1 only
#   runtime         → .NET runtime + base OS (~200MB) — for non-self-contained apps
#   runtime-deps    → JUST the native OS libraries .NET needs (~50MB) — for self-contained apps
#
# Since we used --self-contained true, the .NET runtime is already INSIDE our binary.
# So we only need the OS-level dependencies (glibc, libssl, libicu etc.) → runtime-deps.
#
# "noble-chiseled" = Ubuntu 24.04 but "chiseled" — Microsoft's equivalent of distroless.
# Chiseled = stripped down to only what's needed, no shell, no package manager.
# Same security philosophy as distroless: no shell = much harder to exploit if compromised.
FROM mcr.microsoft.com/dotnet/runtime-deps:10.0.0-noble-chiseled@sha256:b857c8cb8d929183cfe4c6dd9994abba92a2639dd2dbaf06005379f815991604

WORKDIR /app

# Copy the single self-contained binary from builder. That's it.
# No DLL folder, no runtime to install, no package manager. One file.
COPY --from=builder /cartservice .

EXPOSE 7070

# DOTNET_EnableDiagnostics=0 → disables diagnostic features (memory dumps, debugger
# attach, profiling hooks). In a chiseled/distroless image these wouldn't work anyway
# (no shell to attach with), and keeping them enabled wastes resources + attack surface.
#
# ASPNETCORE_HTTP_PORTS=7070 → tells ASP.NET Core which port to listen on.
# ASP.NET has its own internal web server (Kestrel) — this is how you configure it
# without touching code. Equivalent of PORT=7070 in a Node/Express app.
ENV DOTNET_EnableDiagnostics=0 \
    ASPNETCORE_HTTP_PORTS=7070

# Run as non-root user (UID 1000).
# WHY: by default containers run as root — if the container is compromised,
# the attacker has root inside it. Non-root limits the blast radius.
# This is a K8s security best practice — some clusters enforce it via PodSecurityPolicy.
# UID 1000 is a standard unprivileged user. Chiseled images come with this user pre-created.
USER 1000

# Self-contained binary, just like Go. No interpreter prefix needed.
# Unlike Python ("python email_server.py") or Node ("node server.js"),
# the binary IS the executable.
ENTRYPOINT ["/app/cartservice"]