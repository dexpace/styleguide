# 08 — Build & Distribution

The build is part of the codebase. Treat it like one.

## What good looks like

```kotlin
// build.gradle.kts — type-safe DSL (8.1), every version from the catalog (8.2)
plugins {
    alias(libs.plugins.kotlin.jvm)
    `maven-publish`
}

java {
    toolchain { languageVersion = JavaLanguageVersion.of(21) } // 8.3 pin the JDK, 8.10 test on it
}

dependencies {
    implementation(libs.kotlin.coroutines.core) // 8.2 no hard-coded version, 8.4 stdlib left to the plugin
}

kotlin {
    compilerOptions {
        allWarningsAsErrors = true                              // 8.6 warnings as errors
        freeCompilerArgs.addAll("-Xjsr305=strict", "-Xexplicit-api=strict")
    }
}

tasks.withType<AbstractArchiveTask>().configureEach {
    isPreserveFileTimestamps = false                            // 8.7 reproducible artifacts
    isReproducibleFileOrder = true
}
```

This script is Kotlin DSL, not Groovy (8.1); every coordinate resolves through `libs.versions.toml` (8.2) and the stdlib is left to the Kotlin plugin (8.4); the toolchain pins JDK 21 for both compile and test (8.3, 8.10); `allWarningsAsErrors` plus strict-interop and explicit-API flags live in the build, not a CI script (8.6); and the archive tasks strip timestamps for byte-identical output (8.7). Publishing, ABI snapshots, and CI gates (8.5, 8.11, 8.12) extend this same single source of truth.

## Rules

### 8.1 — Gradle Kotlin DSL (`build.gradle.kts`). No Groovy in new projects.

**Reasoning, step by step:**
1. The Kotlin DSL gives type-safe build scripts, IDE auto-completion, and the same language your code is in.
2. Groovy DSL still works but has no static checking — typos and bad APIs fail at build time, not edit time.
3. New projects start in `*.kts`. Existing projects migrate when touched; the migration is mostly mechanical.
4. **Anti-pattern:** mixing `*.gradle` and `*.gradle.kts` files in the same project. Pick one per project.

**Enforcement:** review; CI greps for `*.gradle` (non-`.kts`) build files and fails on any in a new module.

### 8.2 — Version catalogs for dependencies. No hard-coded versions in modules.

**Reasoning, step by step:**
1. Gradle's version catalogs (`gradle/libs.versions.toml`) centralize all dependency versions in one file.
2. Modules reference: `implementation(libs.kotlin.coroutines.core)`. The version lives in `libs.versions.toml`.
3. Upgrading a dependency = changing one file. Without the catalog, you grep across every `build.gradle.kts` and risk inconsistency.
4. Bundle related dependencies: `libs.bundles.spring.web` pulls Spring Boot Starter Web + its companions.

**Enforcement:** review; CI greps module scripts for literal version strings in dependency declarations and fails on any not sourced from `libs.versions.toml`.

### 8.3 — Toolchains pin the JDK. Don't depend on the developer's `JAVA_HOME`.

**Reasoning, step by step:**
1. Gradle's `java { toolchain { languageVersion = JavaLanguageVersion.of(21) } }` tells Gradle to use (and download, if needed) a specific JDK.
2. Without this, every developer might use a different local JDK, leading to "works on my machine" bugs.
3. Pin the JDK version in source. Update deliberately.
4. CI runners and developer machines now compile against the same JDK. Reproducible builds.

**Enforcement:** review; the root build declares a `java`/`kotlin` toolchain and CI runs with `JAVA_HOME` unset to prove the toolchain resolves the JDK.

### 8.4 — `kotlin-stdlib` aligned to the Kotlin compiler version. No mismatches.

**Reasoning, step by step:**
1. The Kotlin compiler bundles a `kotlin-stdlib` version. Importing a different version on the classpath produces silent runtime mismatches.
2. Best practice: let the Kotlin Gradle plugin manage `kotlin-stdlib`. Don't override its version unless you know why.
3. Symptoms of mismatch: `NoSuchMethodError`, `IncompatibleClassChangeError`, behavior diverging from documented API.

**Enforcement:** review; no explicit `kotlin-stdlib` version in any module, and a dependency-resolution check fails the build if a non-plugin-managed stdlib version appears on the classpath.

### 8.5 — `binary-compatibility-validator` in CI for published modules.

**Reasoning, step by step:**
1. The Kotlin team's `binary-compatibility-validator` snapshots the public API of a module to a `*.api` file.
2. Any change to public API shows up as a diff in the snapshot. Reviewers see ABI changes explicitly.
3. CI fails if the snapshot is out of date — forcing developers to either update it (intentional ABI change) or revert (accidental).
4. Required for: published libraries, multi-module repos where module-to-module ABI matters, any module documented as "stable."

**Enforcement:** CI runs `apiCheck`; an out-of-date `*.api` snapshot fails the build.

### 8.6 — Compiler flags: warnings as errors; strict null checks for Java interop.

**Reasoning, step by step:**
1. `-Werror` (warnings as errors) keeps the warning count at zero. Otherwise warnings accumulate and become invisible.
2. `-Xjsr305=strict` (or the JSpecify-equivalent for newer code) makes Java nullability annotations actually enforce.
3. `-Xexplicit-api=strict` requires explicit return types and visibility on every public symbol. Recommended for libraries.
4. `-Xjvm-default=all` (or `all-compatibility`) configures default-method generation; see [ch. 01](./01-java-interop.md), §1.10.
5. Pin these in the build, not in CI scripts. The build is the source of truth.

**Enforcement:** `allWarningsAsErrors = true` and the strict `freeCompilerArgs` set in `compilerOptions`; the compiler fails on any warning.

### 8.7 — Reproducible builds: `archivesBaseName`, no timestamps in artifacts.

**Reasoning, step by step:**
1. A reproducible build produces byte-identical artifacts from the same source. This enables caching, verifies supply-chain integrity, and rules out a class of "the artifact changed but the source didn't" bugs.
2. Gradle: `tasks.withType<AbstractArchiveTask> { isPreserveFileTimestamps = false; isReproducibleFileOrder = true }`.
3. Avoid embedding `Build-Time` headers in JAR manifests; embed `Build-SHA` from `git rev-parse HEAD` instead.
4. CI: cache Gradle outputs by hash of inputs. Reproducible builds make cache hits real.

**Enforcement:** CI builds the artifact twice and diffs the two outputs byte-for-byte; any difference fails the build.

### 8.8 — Shadow jars: deliberate, minimal, with explicit relocations.

**Reasoning, step by step:**
1. A shadow (or "uber") jar bundles dependencies into a single artifact. Useful for executables and FAT-jar-shaped deployments.
2. **Problems:** classpath collisions (two dependencies bring `commons-logging`), license obligations (you're now redistributing every dep), and the artifact grows.
3. Relocate packages of internal dependencies to avoid collisions: `relocate("com.google.protobuf", "internal.protobuf")`. Otherwise consumers' classpaths collide.
4. Prefer NOT shadowing for libraries. Library consumers bring their own deps. Reserve shadow jars for applications/CLI tools.

**Enforcement:** review; the Shadow plugin appears only in application modules, and every bundled internal dependency carries an explicit `relocate`.

### 8.9 — GraalVM native-image config: track `reachability-metadata` alongside source.

**Reasoning, step by step:**
1. Native-image needs reachability metadata (which classes use reflection, JNI, resources). It's third-party JSON or compile-time hints.
2. The metadata lives in `META-INF/native-image/<group>/<artifact>/*.json`. Check into source control; review changes.
3. Spring Boot 3+'s `nativeBuildTools` plugin emits metadata for Spring code automatically. For your own reflective code: hand-write or use `tracing-agent`.
4. Build native and JVM artifacts both in CI. Native-image bugs surface at build time; JVM bugs surface elsewhere. Both are needed.

**Enforcement:** review; `META-INF/native-image/**/*.json` is checked in and diffed in review, and CI runs both the native and JVM build jobs.

### 8.10 — Test on JDK 21+ minimum. Match production runtime exactly.

**Reasoning, step by step:**
1. The JVM version in production is what your code must work on. Test on that JVM, not on whatever the developer happens to have.
2. Gradle's `javaToolchains` lets you compile with JDK 21 and run tests on JDK 21 (or 22, 23, etc.) deliberately.
3. For libraries supporting multiple JDKs: matrix-test on each supported version in CI.
4. **Don't:** compile on JDK 21, test on JDK 17 because the dev machine has 17. The bytecode targets 21 and won't load on 17.

**Enforcement:** the test task binds a JDK 21+ toolchain via `javaToolchains`; CI matrix-tests each supported JDK for multi-version libraries.

### 8.11 — Publishing: signed artifacts, sources jar, javadoc jar.

**Reasoning, step by step:**
1. Maven Central requires: signed artifacts (GPG), sources jar, javadoc/dokka jar. Internal Nexus often does too.
2. Gradle: the `maven-publish` plugin + `signing` plugin handle this. Configure once.
3. **Dokka** generates the documentation jar for Kotlin code. Use it instead of empty placeholder javadoc.
4. CI publishes; developers don't push from laptops. The signing key lives in CI secrets, not on a workstation.

**Enforcement:** review; `maven-publish` + `signing` + Dokka configured, and the publish job runs only in CI with the key from secrets.

### 8.12 — Continuous integration: build, test, lint, license-check, security-scan.

**Reasoning, step by step:**
1. Every PR runs: compile, run all tests, run ktlint, run detekt, run `binary-compatibility-validator` (if applicable), license/vulnerability scan.
2. License compliance via `license-gradle-plugin` or equivalent — fails the build on unapproved licenses.
3. Vulnerability scan via OWASP Dependency-Check, Snyk, or GitHub Dependabot — alerts on known CVEs.
4. **Fail-fast:** if any step fails, the PR is red. No "merge despite test failure" without an explicit override and a follow-up ticket.

**Enforcement:** branch protection requires the full CI gate (compile, test, ktlint, detekt, license, vulnerability scan) green before merge.

## Cross-references

- ABI stability: [ch. 01](./01-java-interop.md), §1.1.
- GraalVM native-image trade-offs: [ch. 07](./07-jvm-performance.md), §7.10.
- Compiler plugins (`kotlin-spring`, `kotlin-jpa`, `kotlin-noarg`): [ch. 03](./03-jvm-frameworks.md), [ch. 04](./04-persistence.md).
