---
name: migrate-spring-boot-4
description: Migrate Spring Boot from version 3.X to 4.X autonomously
---

# Migrate Spring Boot from 3.X to 4.X

Execute this migration autonomously. Make all decisions automatically based on the rules below.

## Overview

This skill migrates a Kotlin/Gradle Spring Boot project from version 3.X to 4.X. Spring Boot 4 includes significant changes including Jakarta EE 11, Java 25 support, Jackson 3.0 migration, and various API changes in Spring Security, Kafka, and other modules.

## ⚠️ IMPORTANT PRE-MIGRATION CHECKS

Before starting the migration, verify the following:

1. **Spring Boot 4 Availability**: Confirm that Spring Boot 4.0.2+ is actually released and stable
2. **Jackson 3.0 Compatibility**: Check Spring Boot 4 release notes for full Jackson 3.0 support status
3. **Dependency Compatibility**: This migration may require Spring Boot 4.0.3+ for full Jackson 3.0 support
4. **Backup**: Ensure you have a clean git state and can revert if needed

**Note**: Spring Boot 4 with Jackson 3.0 may still have compatibility issues. Be prepared to troubleshoot compilation errors related to Jackson package structure.

## Execution Instructions

### 0. Preparation

**Git Branch**
- Ensure you're on the latest `main/master` branch
- Run `git pull` if necessary

**Read Current Configuration**
- Read `build.gradle.kts` to understand current dependencies
- Read `gradle/libs.versions.toml` to find current versions
- Read `.java-version` to check current Java version
- Identify all Spring Boot related dependencies

### 1. Update Java Version

**Update .java-version:**
```
25.0
```

**Update build.gradle.kts:**
```kotlin
java.sourceCompatibility = JavaVersion.VERSION_25
```

### 2. Update gradle/libs.versions.toml

**In [versions] section:**

Update:
```toml
java = "25"
spring-boot = "4.0.2"  # or latest 4.X version
springdoc-openapi = "3.0.1"  # updated for Spring Boot 4
jacoco = "0.8.14"  # updated for Java 25 support
springmockk = "5.0.1"  # updated for Spring Boot 4
sentry-io-plugin = "5.12.2"  # or latest compatible version
springwolf-core = "2.0.0"  # new in Spring Boot 4
springwolf-kafka = "2.0.0"  # updated for Spring Boot 4
```

Add new dependency:
```toml
okio = "3.16.4"  # required by Spring Boot 4
```

Update Docker base image if applicable:
```toml
dockerBaseImage = "your-registry/base/jre25-distroless-nonroot:version"
```

Remove if present:
```toml
spring-security-test = "7.0.2"  # now part of spring-boot-starter-security-test
jackson-version = "2.21.0"  # Jackson is now managed by Spring Boot
springwolf-kafka = "1.21.0"  # old version, update to 2.0.0
```

**In [libraries] section:**

Replace:
```toml
kotlin-stdlib-jdk8 = { group = "org.jetbrains.kotlin", name = "kotlin-stdlib-jdk8", version.ref = "kotlin" }
```

With:
```toml
kotlin-stdlib = { group = "org.jetbrains.kotlin", name = "kotlin-stdlib", version.ref = "kotlin" }
```

Replace standalone dependencies with Spring Boot starters:
```toml
# Replace:
# flyway = { group = "org.flywaydb", name = "flyway-core", version.ref = "flyway" }
# flyway-postgresql = { group = "org.flywaydb", name = "flyway-database-postgresql", version.ref = "flyway" }
# spring-kafka = { group = "org.springframework.kafka", name = "spring-kafka" }

# With:
flyway = { group = "org.springframework.boot", name = "spring-boot-starter-flyway" }
flyway-postgresql = { group = "org.flywaydb", name = "flyway-database-postgresql" }  # no version, managed by Spring Boot
spring-boot-starter-kafka = { group = "org.springframework.boot", name = "spring-boot-starter-kafka" }
spring-boot-restclient = { group = "org.springframework.boot", name = "spring-boot-restclient" }
```

Update Jackson dependencies to Jackson 3.0 (tools.jackson):
```toml
# Replace:
# jackson-module-kotlin = { group = "com.fasterxml.jackson.module", name = "jackson-module-kotlin", version.ref = "jackson-version" }
# jackson-datatype-jsr310 = { group = "com.fasterxml.jackson.datatype", name = "jackson-datatype-jsr310", version.ref = "jackson-version" }

# With (version managed by Spring Boot, remove jsr310 as it's included by default):
jackson-module-kotlin = { group = "tools.jackson.module", name = "jackson-module-kotlin" }
```

Update SpringWolf dependencies:
```toml
# Replace:
# springwolf-kafka = { group = "io.github.springwolf", name = "springwolf-kafka", version.ref = "springwolf-kafka" }

# With (add springwolf-core):
springwolf-core = { group = "io.github.springwolf", name = "springwolf-core", version.ref = "springwolf-core" }
springwolf-kafka = { group = "io.github.springwolf", name = "springwolf-kafka", version.ref = "springwolf-kafka" }
```

Add okio dependency:
```toml
okio = { group = "com.squareup.okio", name = "okio", version.ref = "okio" }
```

Add test dependencies (if not present):
```toml
spring-boot-starter-test = { group = "org.springframework.boot", name = "spring-boot-starter-test" }
spring-boot-resttestclient = { group = "org.springframework.boot", name = "spring-boot-resttestclient" }
spring-boot-starter-webmvc-test = { group = "org.springframework.boot", name = "spring-boot-starter-webmvc-test" }
spring-boot-starter-security-test = { group = "org.springframework.boot", name = "spring-boot-starter-security-test" }
spring-boot-starter-kafka-test = { group = "org.springframework.boot", name = "spring-boot-starter-kafka-test" }
```

Remove if present:
```toml
spring-security-test = { group = "org.springframework.security", name = "spring-security-test", version.ref = "spring-security-test" }
spring-kafka-test = { group = "org.springframework.kafka", name = "spring-kafka-test" }
```

**In [bundles] section:**

Update Kotlin bundle:
```toml
# Replace:
# kotlin = ["kotlin-reflect", "kotlin-stdlib-jdk8", "jackson-module-kotlin", "jackson-datatype-jsr310"]

# With (remove jsr310 as it's included by default in Spring Boot 4):
kotlin = ["kotlin-reflect", "kotlin-stdlib", "jackson-module-kotlin"]
```

Add Spring Boot starter bundle (consolidate individual starter dependencies):
```toml
spring-boot-starter = [
    "spring-boot-starter-web",
    "spring-boot-starter-actuator",
    "spring-boot-starter-security",
    "spring-boot-starter-jooq",
    "spring-boot-starter-kafka",
]
```

Update OpenAPI bundle to include SpringWolf core:
```toml
# Replace:
# openapi = ["springdoc-openapi", "springwolf-kafka"]

# With:
openapi = ["springdoc-openapi", "springwolf-core", "springwolf-kafka"]
```

Add or update test bundle:
```toml
spring-test = [
    "spring-boot-starter-test",
    "spring-boot-restclient",
    "spring-boot-resttestclient",
    "spring-boot-starter-webmvc-test",
    "spring-boot-starter-security-test",
    "spring-boot-starter-kafka-test",
]
```

### 3. Update build.gradle.kts - Dependencies

**Remove deprecated Gradle suppression annotation:**
```kotlin
// Remove this line from the top of the file:
// @Suppress("DSL_SCOPE_VIOLATION") // https://youtrack.jetbrains.com/issue/KTIJ-19369
```

**Replace individual Spring Boot starters with the bundle:**
```kotlin
// Replace:
// implementation(libs.spring.boot.starter.web)
// implementation(libs.spring.boot.starter.actuator)
// implementation(libs.spring.boot.starter.jooq)
// implementation(libs.spring.boot.starter.security)
// implementation(libs.spring.kafka)

// With:
implementation(libs.bundles.spring.boot.starter)
```

**Add okio dependency:**
```kotlin
implementation(libs.okio)
```

**Handle spring-boot-starter-aop:**
```kotlin
// spring-boot-starter-aop may not be available as a standalone dependency in Spring Boot 4
// AOP is now pulled transitively from other Spring Boot starters
// Comment out or remove if present:
// implementation(libs.spring.boot.starter.aop)
```

**Replace Flyway standalone with Spring Boot starter:**
```kotlin
// Keep:
implementation(libs.flyway)  // Now points to spring-boot-starter-flyway
implementation(libs.flyway.postgresql)
```

**Update test dependencies:**
```kotlin
// Replace:
// testImplementation(libs.spring.boot.starter.test)
// testImplementation(libs.spring.security.test)
// testImplementation(libs.spring.kafka.test)

// With:
testImplementation(libs.bundles.spring.test)
```

### 4. Add Jackson Exclusions (Prevent Old Jackson Versions)

**⚠️ CRITICAL: Jackson 3.0 has a SPLIT PACKAGE STRUCTURE**

Jackson 3.0 uses a hybrid package structure:
- **Annotations remain in OLD package**: `com.fasterxml.jackson.annotation.*` (JsonInclude, JsonProperty, etc.)
- **Core/databind moved to NEW package**: `tools.jackson.*` (JsonMapper, JsonSerializer, etc.)

**Add to build.gradle.kts (after dependencies block):**
```kotlin
configurations.all {
    if (name.contains("compile", ignoreCase = true)) {
        // ONLY exclude these. DO NOT exclude: com.fasterxml.jackson.annotation (still used in Jackson 3.0)
        exclude(group = "com.fasterxml.jackson.core", module = "jackson-databind")
        exclude(group = "com.fasterxml.jackson.module", module = "jackson-module-kotlin")
        exclude(group = "com.fasterxml.jackson.datatype", module = "jackson-datatype-jsr310")
    }
}
```

**Why this configuration is critical:**
- Some third-party libraries may still depend on Jackson 2.x databind
- Jackson 3.0 STILL USES `com.fasterxml.jackson.annotation.*` for annotations
- Excluding annotations will cause "Unresolved reference" compilation errors
- This exclusion prevents only the databind/core conflicts, not annotation conflicts

### 5. Update GitHub Workflows

**Update all workflow files in `.github/workflows/` to use Java 25:**

In files like `ci.yaml`, `cd.yaml`, `background-maintenance.yaml`, `manual-deployment.yaml`:

```yaml
# Replace:
# java-version: '21'

# With:
java-version: '25'
```

### 6. Update Jackson/ObjectMapper Configuration

**⚠️ CRITICAL: Jackson 3.0 Package Structure**

Jackson 3.0 uses a SPLIT package structure. Update imports carefully:

**Annotations stay in OLD package:**
```kotlin
import com.fasterxml.jackson.annotation.JsonInclude  // ✓ Keep this
import com.fasterxml.jackson.annotation.JsonProperty  // ✓ Keep this
import com.fasterxml.jackson.annotation.JsonSerialize  // ✓ Keep this
```

**Core/databind move to NEW package:**
```kotlin
// Old imports - REPLACE these:
// import com.fasterxml.jackson.databind.DeserializationFeature
// import com.fasterxml.jackson.databind.ObjectMapper
// import com.fasterxml.jackson.databind.SerializationFeature
// import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule

// New imports - USE these:
import tools.jackson.core.StreamReadFeature
import tools.jackson.core.JsonGenerator
import tools.jackson.core.JsonParser
import tools.jackson.databind.DeserializationFeature
import tools.jackson.databind.DeserializationContext
import tools.jackson.databind.SerializerProvider
import tools.jackson.databind.JsonSerializer
import tools.jackson.databind.cfg.DateTimeFeature
import tools.jackson.databind.json.JsonMapper
import tools.jackson.databind.deser.std.StdDeserializer
import tools.jackson.module.kotlin.KotlinModule
```

**Refactor ObjectMapper bean to use JsonMapper.Builder:**

Old pattern:
```kotlin
@Bean
fun objectMapper(): ObjectMapper {
    val objectMapper = ObjectMapper()
        .registerModule(KotlinModule.Builder().build())
        .registerModule(JavaTimeModule())
        .configure(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS, false)
        .registerKotlinModule()
        .configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false)
    // Additional configuration...
    return objectMapper
}
```

New pattern:
```kotlin
private val kotlinModule = KotlinModule.Builder().build()

@Bean
fun objectMapper(): JsonMapper {
    // Create custom modules if needed
    val customModule = SimpleModule()
        .addDeserializer(YourClass::class.java, YourDeserializer(createMapperBuilder().build()))

    return createMapperBuilder()
        .addModules(customModule, kotlinModule)  // kotlinModule must be added last
        .build()
}

private fun createMapperBuilder(): JsonMapper.Builder =
    JsonMapper.builder()
        .configure(DateTimeFeature.WRITE_DATES_AS_TIMESTAMPS, false)
        .configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false)
        .configure(StreamReadFeature.INCLUDE_SOURCE_IN_LOCATION, true)
        .addMixIn(TargetClass::class.java, MixinClass::class.java)
        .addModules(kotlinModule)
```

Key changes:
- Replace `ObjectMapper()` with `JsonMapper.builder()`
- Replace `SerializationFeature.WRITE_DATES_AS_TIMESTAMPS` with `DateTimeFeature.WRITE_DATES_AS_TIMESTAMPS`
- Remove `JavaTimeModule()` registration (included by default)
- Remove `.registerKotlinModule()` call
- Use builder pattern with `.addModules()` instead of `.registerModule()`
- Add `kotlinModule` last in the module chain

**⚠️ CRITICAL: serializationInclusion API Changed**

The `.serializationInclusion()` method signature changed in Jackson 3.0:

```kotlin
// OLD (Jackson 2.x / Spring Boot 3):
ObjectMapper().setSerializationInclusion(JsonInclude.Include.NON_NULL)

// NEW (Jackson 3.0 / Spring Boot 4):
JsonMapper.builder()
    .serializationInclusion(
        JsonInclude.Value.construct(JsonInclude.Include.NON_NULL, JsonInclude.Include.NON_NULL)
    )
    .build()
```

Note: The method now requires `JsonInclude.Value.construct()` instead of just `JsonInclude.Include`.

### 7. Update Kafka Configuration

**Update package imports:**

In `KafkaConfiguration.kt`:
```kotlin
// Old imports:
// import org.springframework.boot.autoconfigure.kafka.DefaultKafkaConsumerFactoryCustomizer
// import org.springframework.boot.autoconfigure.kafka.DefaultKafkaProducerFactoryCustomizer
// import org.springframework.boot.autoconfigure.kafka.KafkaProperties
// import org.springframework.boot.ssl.SslBundles

// New imports:
import org.springframework.boot.kafka.autoconfigure.DefaultKafkaConsumerFactoryCustomizer
import org.springframework.boot.kafka.autoconfigure.DefaultKafkaProducerFactoryCustomizer
import org.springframework.boot.kafka.autoconfigure.KafkaProperties
```

**Remove SslBundles dependency:**
```kotlin
// Old:
// class KafkaConfiguration(
//     private val properties: KafkaProperties,
//     private val sslBundles: SslBundles,
// )

// New:
class KafkaConfiguration(
    private val properties: KafkaProperties,
)
```

**Update buildProducerProperties() calls:**
```kotlin
// Old:
// val configProps = properties.buildProducerProperties(sslBundles)

// New:
val configProps = properties.buildProducerProperties()
```

**⚠️ IMPORTANT: Kafka Property Assignment May Not Work**

Kotlin property assignment syntax may fail in Spring Boot 4. Use setter methods instead:

```kotlin
// OLD (Spring Boot 3) - May fail in Spring Boot 4:
val factory = ConcurrentKafkaListenerContainerFactory<String, String>()
factory.consumerFactory = consumerFactory  // ❌ May cause "val cannot be reassigned" error

// NEW (Spring Boot 4) - Use setter methods:
val factory = ConcurrentKafkaListenerContainerFactory<String, String>()
factory.setConsumerFactory(consumerFactory)  // ✅ Use explicit setter
factory.setCommonErrorHandler(errorHandler)
```

**Update ExponentialBackOff.maxAttempts type:**

The `maxAttempts` property changed from `Int` to `Long`:

```kotlin
// OLD:
maxAttempts = Int.MAX_VALUE  // ❌ Type mismatch

// NEW:
maxAttempts = Long.MAX_VALUE  // ✅ Correct type
maxAttempts = retryMaxAttempts.toLong()  // ✅ Convert Int to Long
```

### 8. Update Spring Security Configuration

**⚠️ IMPORTANT: Spring Security Package Changes**

UserDetailsServiceAutoConfiguration moved to a new package:

```kotlin
// OLD (Spring Boot 3):
import org.springframework.boot.autoconfigure.security.servlet.UserDetailsServiceAutoConfiguration

// NEW (Spring Boot 4):
import org.springframework.boot.security.autoconfigure.UserDetailsServiceAutoConfiguration
```

**Update AuthorizationManager lambda signatures:**

In security configuration classes (e.g., `BackofficeSecurityConfig.kt`, `CustomerSecurityConfig.kt`):

Old pattern:
```kotlin
private fun backofficeAuthentication(): AuthorizationManager<RequestAuthorizationContext> =
    AuthorizationManager {
        _: Supplier<Authentication>,
        _: RequestAuthorizationContext,
        ->
        AuthorizationDecision(
            SecurityContextHolder.getContext().authentication != null &&
                jwtAuthenticationProvider
                    .authenticate(SecurityContextHolder.getContext().authentication)
                    .isAuthenticated,
        )
    }
```

New pattern:
```kotlin
private fun backofficeAuthentication(): AuthorizationManager<RequestAuthorizationContext> =
    AuthorizationManager { authenticationSupplier: Supplier<out Authentication?>, _: RequestAuthorizationContext ->
        val authentication = authenticationSupplier.get()
        AuthorizationDecision(
            authentication != null &&
                jwtAuthenticationProvider
                    .authenticate(authenticationSupplier.get())
                    .isAuthenticated,
        )
    }
```

**Remove unnecessary imports:**
```kotlin
// Remove:
// import org.springframework.security.core.context.SecurityContextHolder
```

Key changes:
- Update lambda parameter type from `Supplier<Authentication>` to `Supplier<out Authentication?>`
- Use the supplied authentication instead of `SecurityContextHolder.getContext().authentication`
- Extract authentication from supplier with `.get()`

### 9. Update Password Encoder (if applicable)

In custom password encoders (e.g., `SimplePasswordEncoder.kt`):

Old pattern:
```kotlin
import org.springframework.security.crypto.password.PasswordEncoder

class SimplePasswordEncoder : PasswordEncoder {
    override fun encode(rawPassword: CharSequence): String = rawPassword.toString()
    override fun matches(rawPassword: CharSequence, encodedPassword: String): Boolean =
        rawPassword.toString() == encodedPassword
}
```

New pattern (if signature changes are needed):
```kotlin
import org.springframework.security.crypto.password.PasswordEncoder

class SimplePasswordEncoder : PasswordEncoder {
    override fun encode(rawPassword: CharSequence?): String = rawPassword?.toString() ?: ""
    override fun matches(rawPassword: CharSequence?, encodedPassword: String?): Boolean =
        rawPassword?.toString() == encodedPassword
    // Add upgradeEncoding if needed
    override fun upgradeEncoding(encodedPassword: String?): Boolean = false
}
```

### 10. Update JWT Token Services

In JWT token service classes:

**Update imports if using Jackson:**
```kotlin
// Old:
// import com.fasterxml.jackson.databind.ObjectMapper

// New:
import tools.jackson.databind.json.JsonMapper
```

**Update ObjectMapper type:**
```kotlin
// Old:
// class CustomerJwtTokenService(private val objectMapper: ObjectMapper)

// New:
class CustomerJwtTokenService(private val objectMapper: JsonMapper)
```

### 11. Update REST Client Configuration

**⚠️ KNOWN ISSUE: RestTemplateBuilder Package**

RestTemplateBuilder may have package/module issues in Spring Boot 4:

```kotlin
// Expected import:
import org.springframework.boot.web.client.RestTemplateBuilder

// Issue: May cause "Unresolved reference 'client'" compilation error
// This appears to be a Spring Boot 4.0.2 modular structure issue
// The package may not be available at compile-time in some builds

// Workaround options:
// 1. Wait for Spring Boot 4.0.3+ which may fix this
// 2. Migrate to Spring Boot 4's new RestClient API instead:
import org.springframework.web.client.RestClient
```

**Add HTTP URL suppression where applicable:**

In REST client configuration files:
```kotlin
@Suppress("HttpUrlsUsage")
private fun baseUrl(host: String, port: Int, protocol: String): String {
    // Implementation
}
```

### 12. Update Message Converters and Deserializers

**Update Jackson imports in all message converters:**

In files like `*MessageConverter.kt`, `*MessageDeserializer.kt`:
```kotlin
// Old:
// import com.fasterxml.jackson.databind.ObjectMapper

// New:
import tools.jackson.databind.json.JsonMapper
```

**Update constructor parameters:**
```kotlin
// Old:
// class SomeMessageConverter(private val objectMapper: ObjectMapper)

// New:
class SomeMessageConverter(private val objectMapper: JsonMapper)
```

### 13. Update Database Configuration

**Update Flyway configuration (if applicable):**

In `FlywayConfiguration.kt`, update the configuration customizer:

Old pattern:
```kotlin
import org.springframework.boot.autoconfigure.flyway.FlywayConfigurationCustomizer

@Bean
fun configureFlyway(): FlywayConfigurationCustomizer =
    FlywayConfigurationCustomizer {
        with(it) {
            configuration(additionalFlywayConfigurationProperties)
        }
    }
```

New pattern:
```kotlin
import org.flywaydb.core.api.configuration.FluentConfiguration

@Bean
fun configureFlyway(): (FluentConfiguration) -> Unit =
    {
        it.configuration(additionalFlywayConfigurationProperties)
    }
```

Key changes:
- Replace `FlywayConfigurationCustomizer` with lambda function `(FluentConfiguration) -> Unit`
- Change import from `org.springframework.boot.autoconfigure.flyway.FlywayConfigurationCustomizer` to `org.flywaydb.core.api.configuration.FluentConfiguration`
- Simplify lambda by removing `with(it)` wrapper

**Update JOOQ configuration (if applicable):**

In `JooqDslContextConfiguration.kt`:
- Verify JOOQ version compatibility with Spring Boot 4
- Update any custom SQL dialects or configurations

### 14. Update Test Configuration

**Update test class annotations and imports:**

In integration test files:
```kotlin
// Update imports:
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.boot.test.web.client.TestRestTemplate
```

**Update test dependencies usage:**

If using explicit test dependencies:
```kotlin
// Old:
// testImplementation(libs.spring.security.test)

// New (handled by bundles.spring.test):
testImplementation(libs.bundles.spring.test)
```

**Add missing test annotations if needed:**

Some tests may require additional Spring Boot 4 specific configurations:
```kotlin
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureMockMvc
class YourIntegrationTest { /* ... */ }
```

### 15. Update Transaction Configuration

**Check TransactionManager configuration:**

In transaction-related configuration classes:
```kotlin
// Verify TransactionManager bean compatibility
@Bean
fun transactionManager(dataSource: DataSource): PlatformTransactionManager {
    return DataSourceTransactionManager(dataSource)
}
```

### 16. Update Application Configuration

**Move error configuration in application.yaml:**

In `application.yaml` or `application-*.yaml`:
```yaml
# Old (Spring Boot 3):
server:
  error:
    include-message: always

# New (Spring Boot 4):
spring:
  web:
    error:
      include-message: always
```

**Update management endpoints (if applicable):**

In `application.yaml` or `application-*.yaml`:
```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  # Spring Boot 4 may have new endpoint defaults
```

### 17. Add SpringWolf YAML Redirect Controller (if using SpringWolf)

**Create SpringWolfYamlRedirectController.kt:**

SpringWolf 2.0 changes the YAML endpoint behavior. If your application needs YAML format support, create a redirect controller:

```kotlin
package com.your.package.interfaces.rest.springwolf

import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty
import org.springframework.http.HttpHeaders
import org.springframework.http.MediaType.APPLICATION_YAML
import org.springframework.http.MediaType.APPLICATION_YAML_VALUE
import org.springframework.http.ResponseEntity
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RequestMapping
import org.springframework.web.bind.annotation.RestController
import org.springframework.web.client.RestClient
import org.springframework.web.client.toEntity

@RestController
@RequestMapping("/springwolf")
@ConditionalOnProperty(name = ["springwolf.enabled"], havingValue = "true", matchIfMissing = false)
class SpringWolfYamlRedirectController(
    private val restClient: RestClient = RestClient.create(),
) {
    @GetMapping("/docs.yaml", produces = ["application/yaml"])
    fun redirectToYaml(): ResponseEntity<String> {
        val response =
            restClient
                .get()
                .uri("http://localhost:8080/springwolf/docs")
                .header(HttpHeaders.ACCEPT, APPLICATION_YAML_VALUE)
                .retrieve()
                .toEntity<String>()

        return ResponseEntity
            .ok()
            .contentType(APPLICATION_YAML)
            .body(response.body)
    }
}
```

This controller:
- Only activates when `springwolf.enabled=true` in configuration
- Provides a `/springwolf/docs.yaml` endpoint
- Uses `RestClient` to fetch YAML format from the main SpringWolf endpoint
- Properly sets content type headers

## Automatic Decision Rules

Apply these rules without asking:

1. **Use latest stable versions:** Spring Boot 4.0.2 or newer, Java 25
2. **⚠️ CRITICAL: Migrate Jackson carefully:**
   - Annotations stay in `com.fasterxml.jackson.annotation.*` (OLD package)
   - Core/databind move to `tools.jackson.*` (NEW package)
   - Do NOT blindly change all imports!
3. **Consolidate dependencies:** Use Spring Boot starters instead of standalone libraries
4. **Update all test dependencies:** Use the spring-test bundle
5. **Preserve business logic:** Only change framework-related code, not business logic
6. **Update all workflows:** Ensure CI/CD uses Java 25
7. **Fix compilation errors:** Address all API changes systematically
8. **Maintain configuration values:** Keep existing URLs, credentials, feature flags
9. **Comment out spring-boot-starter-aop:** It's pulled transitively in Spring Boot 4
10. **Use setter methods for Kafka:** Property assignment may not work in Spring Boot 4

## Execution Steps (in order)

1. Read `build.gradle.kts`, `gradle/libs.versions.toml`, `.java-version`
2. Update `.java-version` to 25.0
3. Update `gradle/libs.versions.toml` - versions, libraries, bundles (including SpringWolf, okio)
4. Update `build.gradle.kts` - remove DSL_SCOPE_VIOLATION, update Java version, dependencies, and bundles
5. Add Jackson exclusions to build.gradle.kts
6. Update GitHub workflow files to use Java 25
7. Update all Jackson/ObjectMapper configurations
8. Update Kafka configuration files
9. Update Spring Security configuration files
10. Update JWT services and password encoders
11. Update REST client configurations
12. Update all message converters and deserializers
13. Update database configuration files (FlywayConfiguration)
14. Update application.yaml (move server.error to spring.web.error)
15. Create SpringWolfYamlRedirectController if using SpringWolf
16. Update test files and configurations
17. Run `./gradlew clean build` to identify remaining issues
18. Fix any compilation errors
19. Run `./gradlew test` to ensure tests pass
20. Report results

## Common Migration Patterns

### Pattern 1: Jackson ObjectMapper → JsonMapper
```kotlin
// Before:
ObjectMapper().registerModule(module).configure(feature, value)

// After:
JsonMapper.builder().configure(feature, value).addModules(module).build()
```

### Pattern 2: Kafka Factory Setters → Property Access
```kotlin
// Before:
factory.setValueSerializer(serializer)

// After:
factory.valueSerializer = serializer
```

### Pattern 3: Security AuthorizationManager Lambda
```kotlin
// Before:
AuthorizationManager { _: Supplier<Authentication>, _: Context -> /* ... */ }

// After:
AuthorizationManager { authSupplier: Supplier<out Authentication?>, _: Context -> /* ... */ }
```

### Pattern 4: Spring Boot Starters Bundle
```kotlin
// Before (individual starters):
implementation("org.springframework.boot:spring-boot-starter-web")
implementation("org.springframework.boot:spring-boot-starter-actuator")
implementation("org.springframework.boot:spring-boot-starter-security")
implementation("org.springframework.boot:spring-boot-starter-jooq")
implementation("org.springframework.kafka:spring-kafka")

// After (consolidated bundle):
implementation(libs.bundles.spring.boot.starter)
```

### Pattern 5: Flyway Configuration Customizer
```kotlin
// Before:
import org.springframework.boot.autoconfigure.flyway.FlywayConfigurationCustomizer

@Bean
fun configureFlyway(): FlywayConfigurationCustomizer =
    FlywayConfigurationCustomizer { it.configuration(props) }

// After:
import org.flywaydb.core.api.configuration.FluentConfiguration

@Bean
fun configureFlyway(): (FluentConfiguration) -> Unit =
    { it.configuration(props) }
```

### Pattern 6: Application YAML Error Configuration
```yaml
# Before (Spring Boot 3):
server:
  error:
    include-message: always

# After (Spring Boot 4):
spring:
  web:
    error:
      include-message: always
```

## Verification

After migration, verify:
- `./gradlew clean` succeeds
- `./gradlew build` succeeds without warnings
- `./gradlew test` passes all tests
- Application starts successfully
- All actuator endpoints respond correctly
- Kafka consumers and producers work
- Database migrations apply successfully
- Security endpoints authenticate correctly

## Git Commit

Create a descriptive commit message following the repository's conventions:

```
PTECH-XXXXX: Migrate Spring Boot 3.X to 4.X

Upgrade Spring Boot from 3.5.10 to 4.0.2 including all necessary
code changes for compatibility with the new version.

Key changes:
- Update Java version from 21 to 25
- Migrate Jackson from com.fasterxml to tools.jackson (Jackson 3.0)
- Update Kafka configuration for Spring Boot 4 API changes
- Update Spring Security AuthorizationManager lambda signatures
- Replace standalone dependencies with Spring Boot starters
- Consolidate test dependencies into spring-test bundle
- Update ObjectMapper configuration to use JsonMapper.Builder
- Update GitHub workflows to use Java 25
- Fix JWT services and password encoders for new APIs

Benefits:
- Access to latest Spring Boot 4 features and improvements
- Better performance and security
- Java 25 support and optimizations
- Improved dependency management with starters

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
- DO preserve all business logic and configuration values
- DO maintain backward compatibility where possible
- DO address all compilation errors systematically
- DO run tests after migration to verify functionality

## Troubleshooting Common Issues

### Issue: Jackson deserialization errors
**Solution:** Ensure all custom deserializers use `JsonMapper` instead of `ObjectMapper`

### Issue: Kafka consumer/producer not working
**Solution:** Verify property access syntax is used instead of setter methods

### Issue: Security tests failing
**Solution:** Update `AuthorizationManager` lambda signatures and use supplied authentication

### Issue: Tests not finding beans
**Solution:** Ensure test bundle includes all necessary Spring Boot test starters

### Issue: JOOQ code generation fails
**Solution:** Verify JOOQ version is compatible with Java 25 and Spring Boot 4

### Issue: Flyway migrations fail
**Solution:** Check Flyway version compatibility and configuration in Spring Boot 4

### Issue: "Unresolved reference 'annotation'" errors
**Solution:** Jackson 3.0 keeps annotations in `com.fasterxml.jackson.annotation.*` - do NOT change these imports to `tools.jackson`

### Issue: "val cannot be reassigned" in Kafka configuration
**Solution:** Use explicit setter methods (`.setConsumerFactory()`) instead of Kotlin property assignment

### Issue: RestTemplateBuilder not found
**Solution:** This is a known Spring Boot 4.0.2 issue. Consider migrating to the new `RestClient` API or wait for Spring Boot 4.0.3+

### Issue: Type mismatch for maxAttempts (Int vs Long)
**Solution:** Change `Int.MAX_VALUE` to `Long.MAX_VALUE` or use `.toLong()` conversion

### Issue: serializationInclusion method not found
**Solution:** Use `JsonInclude.Value.construct(JsonInclude.Include.NON_NULL, JsonInclude.Include.NON_NULL)` instead of just `JsonInclude.Include.NON_NULL`
