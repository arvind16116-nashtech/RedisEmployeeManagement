# Employee Management Service: Multi-Module Architecture, Artifact Publishing & Distributed Caching

## 1. Architecture & Module Decomposition

This multi-module Maven repository manages employee lifecycle operations using a decoupled domain library, file-backed persistent storage, and an in-memory Redis caching tier.

employee-platform (Root Reactor)
├── employee-library     ──> Standalone domain contracts & models (com.example:employee-library:1.0.0)
└── employee-redis-app   ──> Core REST service consuming library, H2 storage, & Spring Data Redis

### Technical Stack & Dependencies
* **Runtime**: Java 17+
* **Build Tool**: Apache Maven 3.8+
* **Caching Layer**: Redis (running via Docker on port `6379`)
* **Artifact Repository**: Sonatype Nexus OSS (running on port `8081`)
* **Persistence**: File-persisted H2 Database (`./data/employees`)
* **Containerization**: Docker Compose

2. Infrastructure & Artifact Repository Setup
Step 2.1: Spin Up Caching Container
Launch the Redis service in the background (Nexus must already be active on localhost:8081):

docker compose up -d redis
docker compose ps

Step 2.2: Prepare Nexus Hosted Repository
Navigate to the Nexus Web Console at http://localhost:8081.

Ensure a repository named maven-releases exists with the following configuration:

Recipe: maven2 (hosted)

Version Policy: Release

Layout Policy: Strict

Deployment Policy: Allow redeploy

Target Endpoint: http://localhost:8081/repository/maven-releases/

Step 2.3: Configure Local Maven Credentials
Initialize your private settings.xml and inject your Nexus deployment password into the current environment:

Copy-Item settings-nexus.example.xml settings.xml
$securePassword = Read-Host 'Enter Nexus Deployment Password' -AsSecureString
\(env:NEXUS_PASSWORD = [Net.NetworkCredential]::new('',\)securePassword).Password

Note: The server ID configured in settings.xml maps directly to the root pom.xml under ``. Credentials remain session-scoped and excluded from version control.

3. Build Lifecycle & Nexus Deployment Workflow
Step 3.1: Compile & Validate Workspace
Verify compilation, execute tests, and validate modules across the entire Maven reactor:

mvn clean verify

Step 3.2: Deploy Parent & Library to Nexus
Deploy the root parent POM non-recursively, followed by the domain library:

# 1. Publish Parent POM
mvn -N deploy -s settings.xml

# 2. Publish Domain Library
mvn -f employee-library/pom.xml deploy -s settings.xml

Verification: Open Nexus Browse, then expand com/example/employee/ to verify that both employee-platform/1.0.0/ and employee-library/1.0.0/ are indexed.

Step 3.3: Verify Remote Dependency Pull (Cache Busting)
To confirm employee-redis-app resolves the dependency directly from your Nexus server instead of the local .m2 repository:

# Remove local cached artifact
Remove-Item -Recurse -Force "$env:USERPROFILE\.m2\repository\com\example\employee\employee-library\1.0.0"

# Fetch dependency tree via Nexus
mvn -f employee-redis-app/pom.xml dependency:tree -s settings.xml

Confirm that com.example:employee-library:jar:1.0.0 is downloaded and listed in the dependency tree.

4. Running the REST Service
Launch the Spring Boot microservice:

mvn -f employee-redis-app/pom.xml spring-boot:run -s settings.xml

Service URL: http://localhost:8080

H2 Console: http://localhost:8080/h2-console

JDBC URL: jdbc:h2:file:./data/employees

User: sa | Password: (leave empty)

5. API Reference & Contract
Execute requests via Postman, cURL, or the bundled employee-api.http file.

Action,HTTP Verb,Route,Expected Status
Register Employee,POST,/api/employees,201 Created (Location header set)
Fetch All,GET,/api/employees,200 OK
Fetch Single,GET,/api/employees/{id},200 OK (404 Not Found if missing)
Modify Record,PUT,/api/employees/{id},200 OK
Remove Record,DELETE,/api/employees/{id},204 No Content

Sample JSON Payload

{
  "id": null,
  "fullName": "Ada Lovelace",
  "email": "ada.lovelace@example.com",
  "department": "Engineering",
  "salary": 125000.00
}

Validation Constraints
fullName, email, and department are mandatory fields.

email must be structurally valid containing @ and cannot duplicate existing entries.

salary must be numeric and >= 0.0.

6. Distributed Caching & Invalidation Verification
The application integrates Spring Cache backed by Redis using the following strategy:

Individual records use @Cacheable with a configurable TTL (app.cache.employee-ttl, default: 30s).

Writes/updates refresh the individual key via @CachePut.

Deletions purge the entry with @CacheEvict.

The collection cache (GET /api/employees) invalidates completely upon any create, update, or delete event.

Step-by-Step Verification Procedure
Write Record: Send a POST /api/employees and record the generated id.

Cold vs. Warm Reads (@Cacheable):

Send GET /api/employees/{id} -> Fetches from H2 and writes into Redis.

Send GET /api/employees/{id} again -> Returned instantly from Redis.

Verify Key Lifecycle & Expiry (TTL):

# Inspect active keys
docker exec employee-redis redis-cli --scan

# Check remaining TTL (seconds)
docker exec employee-redis redis-cli TTL "employee::"

Observe the counter decrementing. After 30 seconds, the key expires.

Cache Synchronization on Update (@CachePut):

Send a PUT /api/employees/{id} with updated details.

Query GET /api/employees/{id}; the returned payload will reflect changes immediately without waiting for TTL expiration.

Cache Invalidation on Removal (@CacheEvict):

Send a DELETE /api/employees/{id}.

Confirm the key is purged from Redis; a subsequent GET will return 404 Not Found.

(Optional: Use docker exec employee-redis redis-cli FLUSHDB to clear all keys during testing.)

7. Configuration & Environment Overrides

Setting Key,Default,Description
SPRING_DATA_REDIS_HOST,localhost,Redis container host
SPRING_DATA_REDIS_PORT,6379,Redis listener port
APP_CACHE_EMPLOYEE_TTL,PT30S,"Cache duration in ISO-8601 format (e.g., PT1M for 60s)"

To run the stack with persistent RDBMS production instances, replace the H2 dialect, driver, and JDBC URL in application.yml / application.properties with PostgreSQL or MySQL configurations.

To tear down the Redis service:

docker compose down