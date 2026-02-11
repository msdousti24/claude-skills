# Java 25 Migration Skill

This skill migrates a Kotlin/Spring Boot/Gradle project from Java 21 to Java 25.

## Overview

Migrating to Java 25 requires updating multiple configuration files and ensuring all build tools support the new Java version. This skill handles the complete migration process.

## Prerequisites

- Java 25 must be installed on the system
- Project uses Gradle as the build system
- Project uses Kotlin
- Docker must be running (use the relevant docker compose command)

## Migration Steps

### 1. Update Version Catalog (`gradle/libs.versions.toml`)

Update the following versions:

```toml
# Docker Base Image - Change from jre21 to jre25
dockerBaseImage = "758156248519.dkr.ecr.eu-central-1.amazonaws.com/base/jre25-distroless-nonroot:2.0.226"

# Java Version - Change from 21 to 25
java = "25"

# Kotlin Version - Update to 2.3.10 or later (required for Java 25 support)
kotlin = "2.3.10"

# Flyway - Update to version 11.20.3 or later (required for Gradle 9.x compatibility)
# Note that version 12.0.0 seems to be compatible only with Spring Boot 4
flyway = "11.20.3"
flyway-postgresql = "11.20.3"
flyway-gradle-plugin = "11.20.3"
```

### 2. Update Build Script (`build.gradle.kts`)

Change the Java source compatibility:

```kotlin
java.sourceCompatibility = JavaVersion.VERSION_25
```

**Also add the Kotlin annotation compiler flag** to fix warnings about annotation default targets:

In the `kotlin { compilerOptions { ... } }` block, add `-Xannotation-default-target=param-property` to the `freeCompilerArgs`:

```kotlin
kotlin {
    compilerOptions {
        freeCompilerArgs.set(listOf("-Xjsr305=strict", "-Xannotation-default-target=param-property"))
        jvmTarget.set(JvmTarget.fromTarget(libs.versions.java.get()))
    }
}
```

This opts into the new Kotlin behavior where annotations on data class parameters are applied to both the parameter and the backing field, eliminating warnings like:
```
This annotation is currently applied to the value parameter only, but in the future it will also be applied to field.
```

**Fix Spring property placeholder syntax in Kotlin files:**

Replace escaped Spring property placeholders with the new Kotlin 2.x string interpolation prefix syntax:

- **Old syntax:** `"\${property.name}"` (escaped dollar sign)
- **New syntax:** `$$"${property.name}"` (interpolation prefix)

This applies to:
- `@Value` annotations: `@Value($$"${property.name}")`
- `@KafkaListener` parameters: `concurrency = $$"${property.concurrency}"`
- Any other Spring property placeholders in Kotlin strings

**Search pattern:**
```bash
# Find all occurrences in Kotlin files
grep -r '"\$' --include="*.kt" --include="*.kts" .
```

**Example changes:**
```kotlin
// Before
@Value("\${lts.kafka.concurrency}")
private val concurrency: Int

// After
@Value($$"${lts.kafka.concurrency}")
private val concurrency: Int
```

**Important:** Only apply this change to Kotlin files (`*.kt` and `*.kts`). Do not modify Java files or other file types.

### 3. Update Java Version File (`.java-version`)

Change the version:

```
25.0
```

### 4. Update GitHub Workflows (`.github/workflows/`)

**Files to update:**
- `.github/workflows/ci.yaml`
- `.github/workflows/cd.yaml`
- `.github/workflows/background-maintenance.yaml`
- `.github/workflows/manual-deployment.yaml`
- `.github/workflows/codeql.yml`

**Add parameter in each workflow's `with:` section:**
```yaml
jdk-version: '25'
```

### 5. Update README.md

Update all Java version references:

- Change `Java 17+` to `Java 25+`
- Change `temurin17` to `temurin25` in brew install commands
- Update any documentation that mentions Java version requirements

### 6. Update Gradle Wrapper (`gradle/wrapper/gradle-wrapper.properties`)

**Critical:** Gradle must be updated to version 9.3.1 or later for Java 25 support:

```properties
distributionUrl=https\://services.gradle.org/distributions/gradle-9.3.1-bin.zip
```

**Why Gradle 9.x is required:**
- Gradle 8.14.x uses Kotlin 2.0.21 internally, which cannot parse Java 25
- Gradle 9.3.1 includes updated tooling that supports Java 25

## Version Compatibility Matrix

| Component | Minimum Version | Reason |
|-----------|----------------|--------|
| Java | 25.0.1 | Target version |
| Kotlin | 2.3.10 | First version with Java 25 bytecode generation support |
| Gradle | 9.3.1 | Required for Java 25 parsing support |
| Flyway | 12.0.0 | Required for Gradle 9.x compatibility (JavaPluginConvention API removed) |

## Known Issues

### Flyway Gradle Plugin Compatibility

- Flyway versions < 12.0.0 are incompatible with Gradle 9.x
- Error: `org/gradle/api/plugins/JavaPluginConvention` (removed API)
- Solution: Update to Flyway 12.0.0 or later

### Gradle Configuration Cache

- Flyway plugin may have issues with Gradle 9's configuration cache (enabled by default)
- If builds fail, try disabling configuration cache temporarily:
  ```bash
  ./gradlew test --no-configuration-cache
  ```

## Testing the Migration

After making all changes, run:

```bash
# Stop any running Gradle daemons
./gradlew --stop

# Verify Gradle version
./gradlew --version

# Run tests
./gradlew clean test
```

## Files Modified

1. `gradle/libs.versions.toml` - Version catalog
2. `build.gradle.kts` - Java source compatibility and Kotlin compiler options
3. `.java-version` - Local Java version
4. `.github/workflows/ci.yaml` - CI pipeline
5. `.github/workflows/cd.yaml` - CD pipeline
6. `.github/workflows/manual-deployment.yaml` - Manual deployment
7. `README.md` - Documentation
8. `gradle/wrapper/gradle-wrapper.properties` - Gradle version
9. `**/*.kt` and `**/*.kts` - Spring property placeholder syntax (project-wide)

## Complete Example Changes

### Before: `gradle/libs.versions.toml`
```toml
dockerBaseImage = "758156248519.dkr.ecr.eu-central-1.amazonaws.com/base/jre21-distroless-nonroot:2.0.226"
kotlin = "2.2.0"
java = "21"
flyway = "10.22.0"
flyway-postgresql = "10.22.0"
flyway-gradle-plugin = "10.4.0"
```

### After: `gradle/libs.versions.toml`
```toml
dockerBaseImage = "758156248519.dkr.ecr.eu-central-1.amazonaws.com/base/jre25-distroless-nonroot:2.0.226"
kotlin = "2.3.10"
java = "25"
flyway = "12.0.0"
flyway-postgresql = "12.0.0"
flyway-gradle-plugin = "12.0.0"
```

### Before: `build.gradle.kts` (Kotlin compiler options)
```kotlin
kotlin {
    compilerOptions {
        freeCompilerArgs.set(listOf("-Xjsr305=strict"))
        jvmTarget.set(JvmTarget.fromTarget(libs.versions.java.get()))
    }
}
```

### After: `build.gradle.kts` (Kotlin compiler options)
```kotlin
kotlin {
    compilerOptions {
        freeCompilerArgs.set(listOf("-Xjsr305=strict", "-Xannotation-default-target=param-property"))
        jvmTarget.set(JvmTarget.fromTarget(libs.versions.java.get()))
    }
}
```

### Before: Kotlin source files (Spring property placeholders)
```kotlin
@KafkaListener(
    groupId = "my-group",
    topics = ["my-topic"],
    concurrency = "\${lts.kafka.concurrency}",
)
class MyKafkaListener(
    @Value("\${lts.kafka.success-log-interval}")
    successLogInterval: Int,
)
```

### After: Kotlin source files (Spring property placeholders)
```kotlin
@KafkaListener(
    groupId = "my-group",
    topics = ["my-topic"],
    concurrency = $$"${lts.kafka.concurrency}",
)
class MyKafkaListener(
    @Value($$"${lts.kafka.success-log-interval}")
    successLogInterval: Int,
)
```

## Troubleshooting

### Error: "25.0.1" during build
- **Cause:** Gradle version too old (uses Kotlin 2.0.x internally)
- **Solution:** Upgrade to Gradle 9.3.1 or later

### Error: "org/gradle/api/plugins/JavaPluginConvention"
- **Cause:** Flyway plugin version too old for Gradle 9.x
- **Solution:** Upgrade Flyway to 12.0.0 or later

### Warning: "This annotation is currently applied to the value parameter only"
- **Cause:** Kotlin 2.3.10 changes default annotation target behavior
- **Solution:** Add `-Xannotation-default-target=param-property` to Kotlin compiler flags (see Step 2)
- **Details:** This warning appears when annotations like `@Schema` are used on data class parameters. The compiler flag opts into the new behavior where annotations are applied to both parameter and field.

## References

- [Kotlin 2.3.0 Release Notes](https://kotlinlang.org/docs/whatsnew23.html) - Java 25 support
- [Gradle Releases](https://gradle.org/releases/) - Latest Gradle versions
- [Flyway Gradle Plugin](https://plugins.gradle.org/plugin/org.flywaydb.flyway) - Plugin versions

## Success Criteria

The migration is successful when:

1. ✅ `./gradlew --version` shows Java 25.x
2. ✅ `./gradlew build` completes without Java version errors
3. ✅ Tests run (even if some fail due to unrelated issues)
4. ✅ No "JavaPluginConvention" errors
5. ✅ No "25.0.1" parsing errors
6. ✅ No escaped Spring property placeholders (`"\${..."`) in Kotlin files - all converted to `$$"${...}"`
7. ✅ No Kotlin annotation default target warnings
