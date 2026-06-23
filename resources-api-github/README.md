# resources-api

A self-contained Mule 4 REST API that exposes three resource management endpoints over HTTPS. All response data is hardcoded inside DataWeave transforms — no database or external system dependencies required.

---

## Table of Contents

1. [Overview](#overview)
2. [Tech Stack](#tech-stack)
3. [Project Structure](#project-structure)
4. [Configuration](#configuration)
5. [API Reference](#api-reference)
6. [Flows](#flows)
7. [Build & Run](#build--run)
8. [Testing](#testing)
9. [Local Test Commands](#local-test-commands)

---

## Overview

`resources-api` is a lightweight, dependency-free Mule 4 application that demonstrates:

- **HTTP Listener** routing — a single `<http:listener-config>` serving multiple paths and methods
- **DataWeave 2.0** transformations — hardcoded mock data and response building entirely in DataWeave
- **Choice routing** — conditional branching for 200 / 404 responses
- **MUnit testing** — 4 test cases achieving 100% processor coverage
- **HTTPS** — TLS enforced via a bundled self-signed certificate

The application is deployable locally via Anypoint Code Builder and to **CloudHub 2.0** using the included deployment configuration.

---

## Tech Stack

| Component | Version |
|---|---|
| Mule Runtime | 4.11.2 |
| `mule-http-connector` | 1.11.3 |
| `mule-maven-plugin` | 4.10.0 |
| MUnit | 3.7.0 |
| Java | 17 (OpenJDK Temurin) |
| Maven | 3.8+ |

---

## Project Structure

```
resources-api/
├── pom.xml                                              ← Maven build descriptor
├── mule-artifact.json                                   ← Mule runtime metadata (min 4.11.0, Java 17)
├── README.md                                            ← This file
└── src/
    ├── main/
    │   ├── mule/
    │   │   └── resources-api.xml                        ← All flows + global config
    │   └── resources/
    │       ├── config.yaml                              ← Externalised HTTP and TLS properties
    │       ├── keystore.jks                             ← Self-signed TLS certificate (local dev)
    │       └── deploy_ch2.json                          ← CloudHub 2.0 deployment config
    └── test/
        ├── munit/
        │   └── resources-api-test-suite.xml             ← MUnit test suite (4 tests, 100% coverage)
        └── resources/
            ├── config.yaml                              ← Test-classpath copy of config
            └── keystore.jks                             ← Test-classpath copy of keystore
```

---

## Configuration

All runtime values are externalised to `src/main/resources/config.yaml` and referenced in XML via `${property.name}` placeholders. Nothing is hardcoded in the flow configuration.

```yaml
http:
  host: "0.0.0.0"
  port: "8081"

tls:
  keystorePath: "keystore.jks"
  keystorePassword: "mulelocal"
  keyPassword: "mulelocal"
```

### TLS / HTTPS

Mule 4.11+ enforces HTTPS on HTTP Listeners. The application ships with a self-signed RSA-2048 certificate (`keystore.jks`, alias `mule-local`, valid 365 days) for local development. When deployed to CloudHub, the platform provides its own trusted certificate.

> **Note:** The keystore credentials above are suitable for local development only. For production deployments, replace `keystore.jks` and update `config.yaml` with secure credentials managed via Anypoint Secrets Manager.

### `pom.xml` build resources

Both `src/main/resources` and `src/test/resources` are declared with `filtering=false` so that binary files (e.g. `keystore.jks`) are copied to the classpath without modification:

```xml
<build>
    <resources>
        <resource>
            <directory>${project.basedir}/src/main/resources</directory>
            <filtering>false</filtering>
        </resource>
    </resources>
    <testResources>
        <testResource>
            <directory>${project.basedir}/src/test/resources</directory>
            <filtering>false</filtering>
        </testResource>
    </testResources>
</build>
```

---

## API Reference

| Environment | Base URL |
|---|---|
| Local | `https://localhost:8081` |
| CloudHub 2.0 | `https://resources-api-cs00oj.5sc6y6-2.usa-e2.cloudhub.io` |

---

### GET /resources

Returns the complete collection of hardcoded resource objects.

**Request**
```http
GET /resources
Accept: application/json
```

**Response — 200 OK**
```json
[
  { "id": "1", "name": "Resource Alpha", "type": "compute",  "status": "active"   },
  { "id": "2", "name": "Resource Beta",  "type": "storage",  "status": "inactive" },
  { "id": "3", "name": "Resource Gamma", "type": "network",  "status": "active"   }
]
```

---

### GET /resources/{id}

Returns a single resource matching the `id` URI parameter.

**Request**
```http
GET /resources/2
Accept: application/json
```

**Response — 200 OK** (resource found)
```json
{ "id": "2", "name": "Resource Beta", "type": "storage", "status": "inactive" }
```

**Response — 404 Not Found** (resource not found)
```json
{ "error": "Resource not found", "id": "99" }
```

---

### POST /resources

Accepts a JSON payload representing a new resource. Returns a 201 Created response with a success message and an echo of the received body.

**Request**
```http
POST /resources
Content-Type: application/json

{
  "name": "Resource Delta",
  "type": "memory",
  "status": "active"
}
```

**Response — 201 Created**
```json
{
  "message": "Resource created successfully",
  "status": 201,
  "receivedPayload": {
    "name": "Resource Delta",
    "type": "memory",
    "status": "active"
  }
}
```

---

## Flows

All flows are defined in `src/main/mule/resources-api.xml` and share a single global `<http:listener-config>` named `httpListenerConfig`.

### Global configuration elements

| Element | Name | Purpose |
|---|---|---|
| `<configuration-properties>` | — | Loads `config.yaml` into the property placeholder context |
| `<tls:context>` | embedded in listener-connection | Self-signed certificate for local HTTPS |
| `<http:listener-config>` | `httpListenerConfig` | HTTPS listener bound to `0.0.0.0:8081` |

---

### `get-resources-flow`

| Property | Value |
|---|---|
| Trigger | `GET /resources` |
| HTTP status | 200 |

**Logic:**  
A single `<ee:transform>` returns a hardcoded DataWeave array of three resource objects. A `<logger>` records the number of items returned.

**DataWeave script:**
```dataweave
%dw 2.0
output application/json
---
[
    { id: "1", name: "Resource Alpha", "type": "compute",  status: "active"   },
    { id: "2", name: "Resource Beta",  "type": "storage",  status: "inactive" },
    { id: "3", name: "Resource Gamma", "type": "network",  status: "active"   }
]
```

> `"type"` is quoted in DataWeave because `type` is a reserved keyword.

---

### `get-resource-by-id-flow`

| Property | Value |
|---|---|
| Trigger | `GET /resources/{id}` |
| HTTP status | 200 (found) or 404 (not found) |

**Logic:**  
1. An `<ee:transform>` filters the hardcoded array using `attributes.uriParams.id`.
2. A `<choice>` router inspects the filtered result:
   - **Match found** → sets `vars.httpStatus = "200"` and extracts `payload[0]`.
   - **No match** → sets `vars.httpStatus = "404"` and builds a JSON error body.
3. The `<http:listener>` uses `#[vars.httpStatus default 200]` as the dynamic status code.
4. A `<logger>` records the requested ID and final HTTP status.

**Filter DataWeave:**
```dataweave
%dw 2.0
output application/json
var allResources = [
    { id: "1", name: "Resource Alpha", "type": "compute",  status: "active"   },
    { id: "2", name: "Resource Beta",  "type": "storage",  status: "inactive" },
    { id: "3", name: "Resource Gamma", "type": "network",  status: "active"   }
]
---
allResources filter ($.id == attributes.uriParams.id)
```

---

### `post-resource-flow`

| Property | Value |
|---|---|
| Trigger | `POST /resources` |
| HTTP status | 201 |

**Logic:**  
An `<ee:transform>` wraps the incoming `payload` in a structured JSON envelope containing a success message, the status code `201`, and the original request body echoed back as `receivedPayload`. A `<logger>` records the successful processing.

**DataWeave script:**
```dataweave
%dw 2.0
output application/json
---
{
    message:         "Resource created successfully",
    status:          201,
    receivedPayload: payload
}
```

---

## Build & Run

### Prerequisites

- Java 17 (OpenJDK Temurin recommended)
- Apache Maven 3.8+
- Anypoint Code Builder (ACB) **or** a local Mule 4.11.2 EE runtime

### Build the application JAR

```bash
JAVA_HOME=/path/to/java17 mvn clean package
```

Artifact: `target/resources-api-1.0.0-mule-application.jar`

### Run locally in Anypoint Code Builder

1. Open the project in ACB.
2. Click **Run Mule Application** (or use the `Run` panel).
3. The app starts and listens on `https://0.0.0.0:8081`.

---

## Testing

### MUnit test suite

`src/test/munit/resources-api-test-suite.xml` contains 4 test cases that cover all flows and both branches of the `<choice>` router in `get-resource-by-id-flow`.

| # | Test name | Flow | Scenario | Key assertions |
|---|---|---|---|---|
| 1 | `getResourcesFlow-should-return-all-3-resources` | `get-resources-flow` | Returns full array | size=3, ids 1/2/3 present, `type` field non-null |
| 2 | `getResourceById-should-return-200-on-valid-id` | `get-resource-by-id-flow` | id=2 found → 200 | `httpStatus`=200, name=Resource Beta, type=storage, status=inactive |
| 3 | `getResourceById-should-return-404-on-unknown-id` | `get-resource-by-id-flow` | id=99 not found → 404 | `httpStatus`=404, `error`=Resource not found, `id`=99 |
| 4 | `postResourceFlow-should-return-201-on-valid-payload` | `post-resource-flow` | POST body → 201 | message correct, `status`=201, `receivedPayload.name`=Resource Delta |

**Coverage:** 10/10 processors — **100%** across all 3 flows.

### Run tests

```bash
JAVA_HOME=/path/to/java17 mvn test
```

HTML coverage report: `target/site/munit/coverage/`

---

## Local Test Commands

> **Note:** `-k` skips TLS certificate validation for the local self-signed cert. Safe for development only.

```bash
# ── GET all resources ────────────────────────────────────────
curl -sk https://localhost:8081/resources | jq .

# ── GET resource by ID — found ───────────────────────────────
curl -sk https://localhost:8081/resources/1 | jq .
curl -sk https://localhost:8081/resources/2 | jq .
curl -sk https://localhost:8081/resources/3 | jq .

# ── GET resource by ID — not found ───────────────────────────
curl -sk https://localhost:8081/resources/99 | jq .

# ── POST create resource ─────────────────────────────────────
curl -sk -X POST https://localhost:8081/resources \
  -H "Content-Type: application/json" \
  -d '{"name":"Resource Delta","type":"memory","status":"active"}' | jq .

# ── Verify HTTP status codes ─────────────────────────────────
curl -sk -o /dev/null -w "GET /resources      → %{http_code}\n" https://localhost:8081/resources
curl -sk -o /dev/null -w "GET /resources/1    → %{http_code}\n" https://localhost:8081/resources/1
curl -sk -o /dev/null -w "GET /resources/99   → %{http_code}\n" https://localhost:8081/resources/99
curl -sk -o /dev/null -w "POST /resources     → %{http_code}\n" \
  -X POST https://localhost:8081/resources \
  -H "Content-Type: application/json" \
  -d '{"name":"test"}'
```

**Expected status codes:** `200` · `200` · `404` · `201`

---

## Refer to MuleSoft documentation

For authoritative guidance on the components used in this project, see:

- [HTTP Connector](https://docs.mulesoft.com/http-connector/latest/)
- [DataWeave 2.0 Language Guide](https://docs.mulesoft.com/dataweave/latest/)
- [MUnit Testing Framework](https://docs.mulesoft.com/munit/latest/)
- [Mule 4 General Documentation](https://docs.mulesoft.com/general/)