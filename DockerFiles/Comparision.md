
**Full comparison — all five services:**

| | Node | Go | Python | C# | Java |
|---|---|---|---|---|---|
| Build tool | npm | go build | pip | dotnet publish | Gradle |
| Dependency manifest | package.json | go.mod | requirements.txt | .csproj | build.gradle |
| Output type | source + node_modules | single binary | source + site-packages | single binary | folder (script + JARs) |
| Final image base | alpine | distroless/static | alpine | chiseled | alpine |
| Runtime in final image | nodejs | nothing | python | nothing (self-contained) | JRE |
| Cross-compilation | hard | built-in | hard | built-in | N/A (JVM handles it) |
| Non-root user | No | No | No | Yes | No |

**Java's key difference from everything else:** it never produces a self-contained binary. The JVM is always a separate runtime that executes `.jar` files. This is Java's original "write once, run anywhere" design — the JARs are platform-neutral bytecode, the JVM handles the platform specifics. It's the opposite philosophy from Go's "compile once per target platform into a native binary."

**The `downloadRepos` → `installDist` split** is the Java-specific caching pattern to remember. Both steps use gradlew, but they're deliberately separated — first layer caches deps, second layer builds. You'll see this in every well-written Java Dockerfile.
