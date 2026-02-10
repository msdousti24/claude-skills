---
name: migrate-jooq-plugin
description: Migrate from nu.studer.jooq plugin to official org.jooq.jooq-codegen-gradle plugin
---

# Migrate from nu.studer.jooq to Official jOOQ Codegen Plugin

Execute this migration autonomously. Make all decisions automatically based on the rules below.

## Overview

This skill migrates a Gradle/Kotlin project from the older `nu.studer.jooq` plugin to the official `org.jooq.jooq-codegen-gradle` plugin. This modernizes the build configuration and aligns with current jOOQ best practices.

## Execution Instructions

### 0. Preparation

**Git Branch**
- Ensure you're on the latest `main/master` branch
- Run `git pull` if necessary

**Read Current Configuration**
- Read `build.gradle.kts` to understand current jOOQ setup
- Read `gradle/libs.versions.toml` to find jOOQ versions
- Identify the database configuration (PostgreSQL user, password, URL, port, database name)

### 1. Update gradle/libs.versions.toml

**In [versions] section:**

Remove:
```toml
jooq-codegen = "3.19.x"  # or whatever old version
jooq-gradle-plugin = "10.x"


# Jakarta Bind API (needed for JOOQ codegen)
jakarta-bind-api = { strictly = "4.0.4" }


# Jakarta Bind API (needed for JOOQ codegen)
jakarta-bind-api = { group = "jakarta.xml.bind", name = "jakarta.xml.bind-api", version.ref = "jakarta-bind-api" }
```

Add:
```toml
jooq = "3.20.11"  # or latest version
```

**In [libraries] section:**

Add:
```toml
jooq = { group = "org.jooq", name = "jooq", version.ref = "jooq" }
```

**In [plugins] section:**

Replace:
```toml
jooq-gradle-plugin = { id = "nu.studer.jooq", version.ref = "jooq-gradle-plugin" }
```

With:
```toml
jooq-codegen = { id = "org.jooq.jooq-codegen-gradle", version.ref = "jooq" }
```

### 2. Update build.gradle.kts - Remove Old Configuration

**Remove buildscript block with jOOQ workaround:**

Delete the entire block that looks like:
```kotlin
buildscript {
    configurations["classpath"].resolutionStrategy.eachDependency {
        if (requested.group == "org.jooq") {
            useVersion(libs.versions.jooq.codegen.get())
        }
    }
}
```

**Remove old imports:**
```kotlin
import nu.studer.gradle.jooq.JooqGenerate
import org.jooq.meta.jaxb.Logging
import org.jooq.meta.jaxb.Property
```

**Add new import:**
```kotlin
import org.springdoc.openapi.gradle.plugin.OpenApiGeneratorTask
```

### 3. Update build.gradle.kts - Plugin Declaration

**In plugins block, replace:**
```kotlin
alias(libs.plugins.jooq.gradle.plugin)
```

**With:**
```kotlin
alias(libs.plugins.jooq.codegen)
```

### 4. Update build.gradle.kts - Dependencies

**In dependencies block, remove:**
```kotlin
jooqGenerator(libs.postgresql)
jooqGenerator(libs.jakarta.bind.api)
```

**Add:**
```kotlin
jooqCodegen(libs.postgresql)

implementation(libs.jooq)
```

### 5. Update build.gradle.kts - jOOQ Configuration Block

**Replace the entire jooq configuration block.**

**Old format:**
```kotlin
jooq {
    version.set(libs.versions.jooq.codegen)
    edition.set(nu.studer.gradle.jooq.JooqEdition.OSS)

    configurations {
        create("main") {
            generateSchemaSourceOnCompilation.set(true)

            jooqConfiguration.apply {
                logging = Logging.ERROR

                jdbc.apply {
                    driver = "org.postgresql.Driver"
                    url = "jdbc:postgresql://localhost:5431/<database_name>"
                    user = "postgres"
                    password = "password"

                    val sslProperty = Property().apply {
                        key = "ssl"
                        value = "false"
                    }
                    properties.add(sslProperty)
                }

                generator.apply {
                    name = "org.jooq.codegen.KotlinGenerator"
                    database.apply {
                        name = "org.jooq.meta.postgres.PostgresDatabase"
                        inputSchema = "public"
                        forcedTypes.add(
                            org.jooq.meta.jaxb.ForcedType().apply {
                                name = "INSTANT"
                                types = "TIMESTAMPTZ"
                            }
                        )
                    }

                    generate.apply {
                        isDeprecated = false
                        isRecords = true
                        isPojos = false
                        isImmutablePojos = false
                        isPojosAsKotlinDataClasses = false
                        isFluentSetters = true
                        isJavaTimeTypes = true
                        isKotlinNotNullRecordAttributes = true
                    }

                    target.apply {
                        packageName = jooqGeneratedSourcesPackage
                    }
                }
            }
        }
    }
}
```

**New format:**
```kotlin
jooq {
    configuration {
        logging = org.jooq.meta.jaxb.Logging.ERROR
        jdbc {
            driver = "org.postgresql.Driver"
            url = "jdbc:postgresql://localhost:5431/<database_name>"
            user = "postgres"
        }

        generator {
            name = "org.jooq.codegen.KotlinGenerator"
            database {
                name = "org.jooq.meta.postgres.PostgresDatabase"
                inputSchema = "public"
                forcedTypes {
                    forcedType {
                        name = "INSTANT"
                        includeTypes = "TIMESTAMPTZ"
                    }
                }
            }

            generate {
                isDeprecated = false
                isRecords = true
                isPojos = false
                isImmutablePojos = false
                isPojosAsKotlinDataClasses = false
                isFluentSetters = true
                isJavaTimeTypes = true
                isKotlinNotNullRecordAttributes = true
            }

            target {
                packageName = jooqGeneratedSourcesPackage
            }
        }
    }
}
```

**Key changes:**
- Remove `version.set()` and `edition.set()`
- Remove `configurations { create("main") { ... } }` wrapper
- Remove `generateSchemaSourceOnCompilation.set(true)`
- Change `jooqConfiguration.apply` to just `configuration`
- Remove `.apply` from nested blocks (`jdbc`, `generator`, `database`, `generate`, `target`)
- Remove password field (use trust authentication)
- Remove SSL properties handling
- Only add forcedTypes if they exist in the original version
  - Change `forcedTypes.add(...)` to `forcedTypes { forcedType { ... } }`
  - Change `types = "TIMESTAMPTZ"` to `includeTypes = "TIMESTAMPTZ"`

### 6. Update build.gradle.kts - Task Configuration

**Remove old task block:**
```kotlin
tasks.build {
    dependsOn(tasks.withType<JooqGenerate>())
}
```

**Replace old task configuration:**
```kotlin
tasks.withType<JooqGenerate> {
    dependsOn(tasks.flywayMigrate)
    allInputsDeclared.set(true)
}
```

**With new task configuration:**
```kotlin
tasks.jooqCodegen {
    dependsOn(tasks.flywayMigrate)
}
```

### 7. Update build.gradle.kts - KotlinCompile Task

**Update tasks.withType<KotlinCompile> block:**

Add dependencies at the beginning:
```kotlin
tasks.withType<KotlinCompile> {
    dependsOn(tasks.spotlessApply, tasks.jooqCodegen)

    // existing configuration...
    val freeCompilerArguments = listOf("-Xjsr305=strict", "-Xannotation-default-target=param-property")

    compilerOptions {
        freeCompilerArgs.set(freeCompilerArguments)
        jvmTarget.set(JvmTarget.fromTarget(libs.versions.java.get()))
    }

    // Add generated sources to Kotlin source sets
    sourceSets.main {
        kotlin.srcDir("${layout.buildDirectory}/generated-sources/jooq")
    }
}
```

**Remove if present:**
```kotlin
finalizedBy(tasks.spotlessApply)
```

### 8. Simplify Database Credentials (Optional but Recommended)

**Update Flyway configuration:**
```kotlin
flyway {
    url = "jdbc:postgresql://localhost:5431/<database_name>"
    driver = "org.postgresql.Driver"
    user = "postgres"
    // Remove password field
    baselineOnMigrate = true
    locations = arrayOf("filesystem:src/main/resources/db/migration")
}
```

**Update docker-compose.yaml:**
```yaml
services:
  postgres:
    image: postgres:16
    command: ["postgres", "-c", "log_statement=all"]
    environment:
      - POSTGRES_DB=<database_name>
      - POSTGRES_USER=postgres
      - POSTGRES_HOST_AUTH_METHOD=trust
    ports:
      - "5431:5432"
```

**Update application-local.yaml:**
```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5431/<database_name>
    username: postgres
    # Remove password field
```

**Update application-test.yaml:**
```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5431/<database_name>
    username: postgres
    # Remove password field
```

### 9. Type Safety in OpenAPI Tasks (Optional)

**If you have OpenAPI generator tasks, update them:**

Replace:
```kotlin
tasks.register<org.springdoc.openapi.gradle.plugin.OpenApiGeneratorTask>("generateSyncApiDocs") {
    // configuration
}
```

With:
```kotlin
tasks.register<OpenApiGeneratorTask>("generateSyncApiDocs") {
    // configuration
}
```

## Automatic Decision Rules

Apply these rules without asking:

1. **Extract database configuration:** From existing jOOQ config (URL, port, database name, user)
2. **Keep package names:** Use existing `jooqGeneratedSourcesPackage` value
3. **Keep generator settings:** Preserve all `generate` block settings
4. **Use latest jOOQ version:** 3.20.11 or newer
5. **Simplify auth:** Use `postgres` user with trust auth for local/test environments
6. **Preserve forced types:** Keep existing type mappings, just update syntax

## Execution Steps (in order)

1. Read `build.gradle.kts` to understand current jOOQ configuration
2. Read `gradle/libs.versions.toml` to find version references
3. Extract database configuration (URL, port, database name, user, password)
4. Update `gradle/libs.versions.toml`
5. Update `build.gradle.kts` - remove old imports and buildscript
6. Update `build.gradle.kts` - change plugin declaration
7. Update `build.gradle.kts` - update dependencies
8. Update `build.gradle.kts` - replace jOOQ configuration block
9. Update `build.gradle.kts` - update task configuration
10. Update `build.gradle.kts` - update KotlinCompile task
11. Optionally simplify database credentials in docker-compose, application configs, and Flyway
12. Run `./gradlew clean build` to verify migration
13. Report results

## Verification

After migration, verify:
- `./gradlew clean` succeeds
- `./gradlew build` succeeds
- jOOQ code generation produces files in `build/generated-sources/jooq`
- Tests pass with `./gradlew test`

## Git Commit

Create a descriptive commit message:

```
Migrate from nu.studer.jooq to official jOOQ plugin

Replace the older nu.studer.jooq Gradle plugin with the official
org.jooq.jooq-codegen-gradle plugin. This modernizes the build
configuration and aligns with current jOOQ best practices.

Key changes:
- Update gradle/libs.versions.toml with new plugin reference
- Replace plugin declaration in build.gradle.kts
- Simplify jOOQ configuration block (remove version, edition, nested configs)
- Update task dependencies (jooqCodegen instead of JooqGenerate)
- Add explicit jOOQ implementation dependency
- Update Kotlin compile task to include generated sources
- Simplify database credentials to use trust authentication
- Add Jackson exclusions for Spring Boot compatibility

Benefits:
- Uses official jOOQ plugin maintained by jOOQ team
- Simpler configuration with modern Gradle DSL
- Better IDE integration with generated sources
- Reduced buildscript complexity

Build and all tests pass successfully.

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>
```

## Important Constraints

- DO NOT ask for confirmation
- DO NOT ask clarifying questions
- DO make sensible automatic decisions based on existing configuration
- DO report progress and final status
- DO verify build passes at the end
- DO NOT push commits automatically (wait for user)
- DO preserve all existing jOOQ generator settings
- DO maintain backward compatibility with generated code
