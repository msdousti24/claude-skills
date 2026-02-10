---
name: migrate-testcontainers
description: Migrate a Spring Boot/Gradle project from TestContainers to docker-compose autonomously
---

# Autonomous TestContainers to docker-compose Migration

Execute this migration completely autonomously. Do not ask for confirmation or clarification. Make all decisions automatically based on the rules below.

## Execution Instructions

Migrate this Spring Boot/Gradle project from TestContainers to docker-compose for both JOOQ code generation and integration tests.

### 0. Preparation 
**Git Branch**
Make sure you are on the latest `main/master` git branch.
Run `git pull` if necessary

**Existing Docker**
Run the following command to shutdown and remove any existing docker containers:
```bash
docker ps -aq | xargs -r docker rm -f
```

### 1. Update docker-compose.yaml

**File:** `docker/docker-compose.yaml` or `docker-compose.yaml`

Transform the docker-compose file:
- **PostgreSQL**:
  - Change user to `postgres`
  - Use `POSTGRES_HOST_AUTH_METHOD=trust` (no password)
  - Add command: `["postgres", "-c", "log_statement=all"]`
  - Add healthcheck:
    ```yaml
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres -d <database_name>"]
      interval: 5s
      timeout: 5s
      retries: 5
    ```
- **Kafka**:
  - If using Zookeeper: migrate to KRaft mode (remove Zookeeper service entirely)
  - Add KRaft configuration:
    ```yaml
    KAFKA_BROKER_ID: 1
    KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: 'CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT'
    KAFKA_ADVERTISED_LISTENERS: 'PLAINTEXT://kafka:29092,PLAINTEXT_HOST://localhost:9092'
    KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
    KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
    KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
    KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
    KAFKA_JMX_PORT: 9201
    KAFKA_JMX_HOSTNAME: localhost
    KAFKA_PROCESS_ROLES: 'broker,controller'
    KAFKA_NODE_ID: 1
    KAFKA_CONTROLLER_QUORUM_VOTERS: '1@kafka:29093'
    KAFKA_LISTENERS: 'PLAINTEXT://kafka:29092,CONTROLLER://kafka:29093,PLAINTEXT_HOST://0.0.0.0:9092'
    KAFKA_INTER_BROKER_LISTENER_NAME: 'PLAINTEXT'
    KAFKA_CONTROLLER_LISTENER_NAMES: 'CONTROLLER'
    KAFKA_LOG_DIRS: '/tmp/kraft-combined-logs'
    CLUSTER_ID: "YWJhMGUzNjAxZTA0NDFiMz"
    ```
  - Add healthcheck:
    ```yaml
    test: nc -z localhost 9092 || exit -1
    interval: 5s
    timeout: 10s
    retries: 10
    ```
- Keep existing image versions and ports
- Remove attribute `version` because it is deprecated

- **RabbitMQ**:
  - Only if used in the project
  - Example setup
  ```yaml
  rabbitmq:
    container_name: "<project-name>-rabbitmq"
    image: rabbitmq:3.8.19-management
    volumes:
      - ./rabbitmq/definitions.json:/etc/rabbitmq/definitions.json
    ports:
      - "5671:5672"
      - "15671:15672"
    healthcheck:
      test: rabbitmq-diagnostics -q ping
      interval: 5s
      timeout: 10s
      retries: 10
  ```

### 2. Update build.gradle.kts

**Remove:**
- TestContainers imports: `import org.testcontainers.*`
- buildscript block with TestContainers dependencies
- PostgreSQL container initialization code
- Container stop/cleanup code in doLast blocks

**Add import:**
```kotlin
import java.io.ByteArrayOutputStream
```

**Update Flyway configuration to:**
```kotlin
flyway {
    url = "jdbc:postgresql://localhost:<port>/<database_name>"
    driver = "org.postgresql.Driver"
    user = "postgres"
    baselineOnMigrate = true
    locations = arrayOf("filesystem:src/main/resources/db/migration")
}
```

**Update JOOQ jdbc configuration to:**
```kotlin
jdbc.apply {
    driver = "org.postgresql.Driver"
    url = "jdbc:postgresql://localhost:<port>/<database_name>"
    user = "postgres"
}
```

### 3. Update gradle/libs.versions.toml

**Remove from [versions]:**
```toml
testcontainers = { ... }
```

**Remove from [libraries]:**
All testcontainers-* entries:
```toml
testcontainers-dependencies-bom = { ... }
testcontainers = { ... }
testcontainers-jdbc = { ... }
testcontainers-postgresql = { ... }
testcontainers-kafka = { ... }
```

**Remove from [bundles]:**
```toml
testcontainers = [...]
```

**Remove from build.gradle.kts dependencies:**
```kotlin
testImplementation(platform(libs.testcontainers.dependencies.bom))
testImplementation(libs.bundles.testcontainers)
```

### 4. Replace TestContainers test configuration

**Delete entire directory:**
`src/test/kotlin/**/testing/libraries/testcontainers/`

**Update or create:** `src/test/resources/application-test.yaml`

Add or update spring configuration:
```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:<port>/<database_name>
    username: postgres

  kafka:
    bootstrap-servers: localhost:9092
```

### 5. Update test annotations

**Find and update files matching:** `**/IntegrationTest.kt`, `**/NonTransactionalIntegrationTest.kt`

**Remove:**
- Imports for testcontainers configuration classes
- `@ContextConfiguration(initializers = [...])` entire annotation line

Configuration now comes from application-test.yaml automatically.

### 6. Update application-local.yaml

**File:** `src/main/resources/application-local.yaml`

Update spring.datasource section:
```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:<port>/<database_name>
    username: postgres
```

### 7. Handle JUnit 4 architecture test

**If file exists:** `**/Junit5ArchUnitTest.kt` or similar that checks for JUnit 4 usage

**If it contains:** `org.junit.Test::class.java`

**Change to:**
```kotlin
.notBeAnnotatedWith("org.junit.Test")  // String-based check
```

**If any file has:** `import org.junit.Assert.*`

**Change to:**
```kotlin
import org.junit.jupiter.api.Assertions.*
```

### 8. Update GitHub workflows

**Files to update:**
- `.github/workflows/ci.yaml`
- `.github/workflows/cd.yaml`
- `.github/workflows/background-maintenance.yaml`
- `.github/workflows/manual-deployment.yaml`

**Add parameter in each workflow's `with:` section:**
```yaml
docker-compose-path: './docker/docker-compose.yaml'
```

### 9. Update README.md

**After the "Run tests" section, add:**

````markdown
## Docker Containers

This project uses docker-compose to manage PostgreSQL and Kafka containers for both JOOQ code generation and integration tests.

**Start containers before building:**
```shell
docker-compose -f ./docker/docker-compose.yaml up -d
```

**Check container status:**
```shell
docker ps | grep <project-name>
```

**Stop containers:**
```shell
docker-compose -f ./docker/docker-compose.yaml down
```

**Stop and remove volumes:**
```shell
docker-compose -f ./docker/docker-compose.yaml down -v
```

**Note:** Containers remain running between builds for better performance. The build will fail with a clear error message if containers aren't running.
````

**Update "Run tests" section start:**
```markdown
**Note**: Docker needs to be running and containers must be started with `docker-compose -f ./docker/docker-compose.yaml up -d` before running tests.
```

**Update features section** - replace TestContainers mentions with docker-compose:
- "via TestContainers" → "via docker-compose"

## Automatic Decision Rules

Apply these rules without asking:

1. **Extract database name:** From existing docker-compose or TestContainers config
2. **Extract PostgreSQL port:** From docker-compose (usually 5431)
3. **Extract Kafka port:** From docker-compose (usually 9092)
4. **Container name prefix:** Use project name from settings.gradle.kts or directory name
5. **Use trust auth:** Always for local/test
6. **Keep image versions:** Use existing versions from docker-compose
7. **Use Java version:** Check build.gradle.kts jdk-version or project Java version

## Execution Steps (in order)

1. Read docker-compose.yaml to extract: database name, ports, container names, image versions
2. Read build.gradle.kts to find TestContainers usage patterns
3. Update docker-compose.yaml
4. Update build.gradle.kts
5. Update gradle/libs.versions.toml
6. Delete testcontainers directory
7. Update application-test.yaml
8. Update test annotation files
9. Update application-local.yaml
10. Fix JUnit 4 if present
11. Update all GitHub workflows
12. Update README.md
13. Start containers: `docker-compose -f ./docker/docker-compose.yaml up -d`
14. Run `./gradlew clean`. Do not use `--no-daemon`
15. Run tests: `./gradlew test`. Do not use `--no-daemon`
16. Report results

## Git Commits

Create a branch, and commit all the changes in one go. Add a descriptive commit message like this:


**Commit 1:**
```
Remove TestContainers in favor of docker-compose

Replace TestContainers with static docker-compose configuration for both
JOOQ code generation and integration tests. This change simplifies the
architecture and improves build performance by keeping containers running
between builds.

Key changes:
- Remove TestContainers dependencies from build.gradle.kts and libs.versions.toml
- Update docker-compose.yaml to use PostgreSQL trust auth and Kafka KRaft mode
- Replace dynamic container configuration with static connection strings
- Configure tests via application-test.yaml instead of initializer classes
- Remove JUnit 4 dependencies and convert remaining JUnit 4 assertions to JUnit 5
- Add docker container validation task to fail fast if containers aren't running
- Update documentation (README.md) with docker-compose instructions

Benefits:
- Faster builds: ~10-20s vs ~30-60s (containers remain running)
- Simpler architecture: static connection strings, no dynamic management
- Better debugging: persistent containers with docker logs
- Reduced dependencies: no TestContainers library

All tests pass successfully.

Add docker-compose-path to GitHub workflows

Add docker-compose-path parameter to all workflows that run tests or builds
to ensure PostgreSQL and Kafka containers are started via docker-compose
before running the application.

Updated workflows:
- ci.yaml - Add docker-compose-path for CI tests
- cd.yaml - Add docker-compose-path for CD builds
- background-maintenance.yaml - Add docker-compose-path for scheduled tests
- manual-deployment.yaml - Add docker-compose-path for manual deployments

This ensures the reusable workflows automatically start the docker-compose
containers before running tests, replacing the previous TestContainers
approach with persistent containers.

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>
```


## Important Constraints

- DO NOT ask for confirmation
- DO NOT ask clarifying questions
- DO NOT update CLAUDE.md (keep PR focused)
- DO make sensible automatic decisions
- DO report progress and final status
- DO verify tests pass at the end
- DO NOT push commits automatically (wait for user)
