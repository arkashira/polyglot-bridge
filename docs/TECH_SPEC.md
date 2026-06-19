# TECH_SPEC.md
**Project:** polyglot-bridge  
**Owner:** AxentX – Multi‑Language Integration Platform  
**Status:** Draft (ready for engineering review)  
**Last Updated:** 2026‑06‑19  

---  

## 1. Overview  

polyglot-bridge is a **runtime‑agnostic integration layer** that allows services, libraries, and scripts written in different programming languages to invoke each other as if they were native calls. It provides:

* **Language‑agnostic RPC** over a lightweight binary protocol.  
* **Automatic type marshaling** using a shared schema definition (JSON‑Schema + protobuf).  
* **Zero‑copy data transfer** for large buffers via shared memory (POSIX shm / Windows named shared memory).  
* **Pluggable transport** (Unix domain sockets, TCP, TLS, gRPC, WebSockets).  
* **Dynamic discovery** of remote endpoints via a built‑in service registry.  

The platform is built on top of **vLLM** for high‑throughput inference (optional) and **SGLang** for structured generation of request/response payloads when needed (e.g., AI‑assisted code translation).  

---  

## 2. Architecture  

```
+-------------------+        +-------------------+        +-------------------+
|   Language A      |        |   Language B      |        |   Language C      |
| (Python, Node…)  |        | (Rust, Go…)       |        | (Java, .NET…)     |
+--------+----------+        +--------+----------+        +--------+----------+
         |                           |                           |
         |   polyglot-bridge SDK    |   polyglot-bridge SDK    |
         +-----------+---------------+-----------+---------------+
                     |                           |
               +-----v-----+               +-----v-----+
               |  Bridge   |   Transport   |  Bridge   |
               |  Core     |<==============>|  Core     |
               +-----+-----+   (pluggable) +-----+-----+
                     |                           |
               +-----v---------------------------v-----+
               |          Service Registry (etcd)      |
               +---------------------------------------+
```

* **Bridge Core** – a native C++ runtime (`libpolybridge.so`) that implements the protocol, memory management, and transport abstraction.  
* **Language SDKs** – thin wrappers (Python, Node.js, Rust, Go, Java, .NET) that expose a **`BridgeClient`** class with async/await APIs.  
* **Service Registry** – optional etcd cluster (or embedded mode) for endpoint discovery and health‑checking.  

---  

## 3. Core Components  

| Component | Description | Language / Tech |
|-----------|-------------|-----------------|
| **Bridge Core** | High‑performance binary protocol engine, zero‑copy shared‑memory buffers, transport adapters. | C++17, libuv, protobuf, flatbuffers |
| **Protocol Layer** | Message framing, versioning, authentication (JWT/HMAC), compression (zstd). | Protobuf v3 |
| **Transport Adapters** | Unix domain socket, TCP (TLS), gRPC‑compatible, WebSocket (WSS). | libuv + OpenSSL |
| **Schema Registry** | Stores JSON‑Schema / protobuf definitions for each service method. | etcd (v3 API) |
| **SDK Generators** | Generates language‑specific client stubs from schema. | Python (Jinja2), Node (Handlebars), Rust (proc‑macro) |
| **Discovery Agent** (optional) | Periodic health‑check pings, updates registry. | Go (goroutine) |
| **AI Assist Module** (optional) | Uses vLLM + SGLang to auto‑translate payloads when schema mismatches are detected. | Python, vLLM, SGLang |

---  

## 4. Data Model  

### 4.1 Service Definition  

```json
{
  "service": "com.example.math",
  "version": "1.2.0",
  "methods": [
    {
      "name": "add",
      "request": "AddRequest",
      "response": "AddResponse",
      "streaming": false
    },
    {
      "name": "streamPrimes",
      "request": "PrimeStreamRequest",
      "response": "PrimeNumber",
      "streaming": true
    }
  ],
  "schemas": {
    "AddRequest": { "$ref": "schemas/AddRequest.json" },
    "AddResponse": { "$ref": "schemas/AddResponse.json" },
    "...": {}
  }
}
```

* Schemas are stored as **JSON‑Schema** files and compiled to **protobuf** for the core engine.  

### 4.2 Message Envelope  

| Field | Type | Description |
|-------|------|-------------|
| `msg_id` | uint64 | Unique per‑connection identifier |
| `service` | string | Target service name |
| `method` | string | Method name |
| `payload` | bytes | Serialized protobuf payload |
| `metadata` | map<string,string> | Optional headers (auth, tracing) |
| `flags` | uint32 | Bitmask (compression, streaming, error) |

---  

## 5. Key APIs / Interfaces  

### 5.1 BridgeClient (language‑agnostic pseudo‑signature)

```python
class BridgeClient:
    async def connect(self, endpoint: str, auth_token: str = None) -> None: ...
    async def invoke(self, service: str, method: str, request: Any) -> Any: ...
    async def stream(self, service: str, method: str, request: Any) -> AsyncIterator[Any]: ...
    async def close(self) -> None: ...
```

* All SDKs expose the same async API; underlying implementation forwards to the native core via **CFFI** (Python), **N-API** (Node), **JNI** (Java), **P/Invoke** (C#), etc.  

### 5.2 Service Registration (via Registry API)

```
PUT /v1/services/{service_name}
Body: { "address": "unix:///tmp/bridge.sock", "metadata": {...} }
```

* Returns `201 Created` with health‑check URL.  

### 5.3 Health‑Check Endpoint  

```
GET /healthz
Response: { "status": "ok", "uptime_seconds": 12345 }
```

---  

## 6. Technology Stack  

| Layer | Choice | Rationale |
|-------|--------|-----------|
| Core Runtime | **C++17**, libuv, protobuf, flatbuffers | Low latency, cross‑platform, zero‑copy |
| Transport | libuv + OpenSSL | Unified event loop, TLS support |
| Schema | **JSON‑Schema** + **protobuf** | Human‑readable + binary efficiency |
| Registry | **etcd v3** (embedded or external) | Strong consistency, watch API |
| SDK Generation | **Jinja2** (Python), **Handlebars** (Node), **proc‑macro** (Rust) | Single source of truth from schema |
| Compression | **zstd** (level 3 default) | High speed, good ratio |
| Authentication | **JWT** (HS256) + optional **mutual TLS** | Flexible per‑deployment |
| AI Assist (optional) | **vLLM** for inference, **SGLang** for structured generation | Auto‑translation of mismatched schemas |
| CI/CD | GitHub Actions, Docker, Helm (K8s) | Standard AxentX pipeline |

---  

## 7. Dependencies  

| Dependency | Version | License |
|------------|---------|---------|
| libuv | 1.44.2 | MIT |
| protobuf | 3.21.12 | BSD‑3 |
| flatbuffers | 23.5.26 | Apache‑2.0 |
| zstd | 1.5.5 | BSD‑3 |
| OpenSSL | 3.1.4 | Apache‑2.0 |
| etcd | 3.5.12 | Apache‑2.0 |
| vLLM (optional) | 0.3.0 | Apache‑2.0 |
| SGLang (optional) | 0.2.1 | Apache‑2.0 |
| Python SDK | 3.11+ | MIT |
| Node SDK | 20+ | MIT |
| Rust SDK | 1.70+ | MIT/Apache‑2.0 |
| Go SDK | 1.22+ | BSD‑3 |
| Java SDK | 17+ | Apache‑2.0 |
| .NET SDK | 8.0+ | MIT |

---  

## 8. Deployment Model  

### 8.1 Container Image  

* **Base:** `ubuntu:22.04`  
* **Layers:**  
  1. Core runtime (`/opt/polybridge/libpolybridge.so`)  
  2. SDK packages (`/opt/polybridge/sdk/*`)  
  3. Optional AI Assist side‑car (`vllm` + `sglang`)  

**Dockerfile (excerpt)**  

```dockerfile
FROM ubuntu:22.04 AS builder
RUN apt-get update && apt-get install -y build-essential cmake git libssl-dev libuv1-dev zstd libprotobuf-dev protobuf-compiler
WORKDIR /src
COPY . .
RUN cmake -S . -B build -DCMAKE_BUILD_TYPE=Release && cmake --build build --target all

FROM ubuntu:22.04
RUN apt-get update && apt-get install -y libssl3 libuv1 zstd libprotobuf23
COPY --from=builder /src/build/libpolybridge.so /opt/polybridge/
COPY sdk/python /opt/polybridge/sdk/python
ENV PATH="/opt/polybridge/sdk/python/bin:${PATH}"
ENTRYPOINT ["polybridge-daemon"]
```

### 8.2 Kubernetes Helm Chart  

* **Chart name:** `polyglot-bridge`  
* **Values:** `replicaCount`, `service.type` (ClusterIP/NodePort), `registry.enabled` (embedded etcd), `aiAssist.enabled`.  
* **Pod spec:**  
  * `securityContext.runAsNonRoot: true`  
  * `resources.limits.cpu: "500m"` `memory: "256Mi"` (core)  
  * Optional side‑car container `vllm` with GPU resource request when `aiAssist.enabled`.  

### 8.3 Scaling & High Availability  

* **Stateless core** – horizontal scaling via load‑balanced TCP/Unix socket front‑end (Envoy).  
* **Registry** – etcd cluster (odd number of nodes).  
* **Shared‑memory buffers** are per‑process; cross‑process zero‑copy works only within the same node. For cross‑node large payloads, fallback to TLS‑encrypted TCP with chunked streaming.  

---  

## 9. Security Considerations  

1. **Authentication** – JWT validated at the core; optional mTLS for intra‑cluster traffic.  
2. **Authorization** – ACL stored in registry (`service -> allowed_clients`). Core checks before dispatch.  
3. **Input Validation** – All inbound payloads validated against compiled protobuf schema; malformed messages cause immediate connection termination.  
4. **Isolation** – Each Bridge instance runs under a dedicated Linux user; shared‑memory segments are created with `0600` permissions.  
5. **Audit Logging** – Configurable JSON log (`/var/log/polybridge/audit.log`) with request ID, client ID, method, latency.  

---  

## 10. Testing Strategy  

| Test Type | Tool | Coverage Goal |
|-----------|------|---------------|
| Unit (C++) | GoogleTest + GoogleMock | 90% |
| Integration (SDK ↔ Core) | pytest (Python), jest (Node), cargo test (Rust) | 80% |
| End‑to‑End (multi‑lang) | Test harness launching Python ↔ Rust services via Docker Compose | 75% |
| Performance | `wrk2` for throughput, `perf` for latency, zero‑copy benchmark | 100k req/s per core on 2‑vCPU |
| Security | OWASP ZAP (fuzzing), JWT tamper tests | No critical findings |
| AI Assist validation | Compare auto‑translated payloads vs hand‑crafted (diff < 2%) | N/A (optional) |

---  

## 11. Roadmap (post‑MVP)

| Milestone | Target | Description |
|-----------|--------|-------------|
| **MVP** | Q3 2026 | Core runtime, Python & Rust SDKs, TCP + Unix socket transports, embedded etcd registry. |
| **Full Language Coverage** | Q4 2026 | Add Node, Java, .NET SDKs, WebSocket transport. |
| **AI Assist** | Q1 2027 | Integrated vLLM + SGLang for on‑the‑fly schema translation. |
| **Observability Suite** | Q2 2027 | OpenTelemetry instrumentation, Grafana dashboards. |
| **Enterprise Hardened** | Q3 2027 | Role‑based ACL, audit log aggregation, FIPS‑compliant TLS. |

---  

## 12. Glossary  

* **Bridge Core** – Native C++ engine handling protocol, transport, and memory.  
* **SDK** – Language‑specific wrapper exposing `BridgeClient`.  
* **Registry** – Service discovery component (etcd).  
* **Zero‑Copy** – Transfer of large buffers via shared memory without serialization.  
* **AI Assist** – Optional module that uses LLM inference to reconcile schema mismatches.  

---  

*Prepared by:* Senior Product/Engineering Lead – AxentX  
*Document ID:* POLYGLOT‑BRIDGE‑TECH‑SPEC‑2026‑06‑19
