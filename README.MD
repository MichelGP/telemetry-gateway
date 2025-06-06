# Telemetry Gateway (OpenTelemetry Collector)

This repository contains a Docker Compose setup for running an OpenTelemetry Collector as a centralized "telemetry-gateway". It aggregates metrics, logs, and traces from applications within a secure compartment and forwards them to a central Grafana LGTM stack (Loki, Mimir, Tempo).

## Table of Contents

- [Telemetry Gateway (OpenTelemetry Collector)](#telemetry-gateway-opentelemetry-collector)
  - [Table of Contents](#table-of-contents)
  - [Overview](#overview)
  - [Architecture](#architecture)
  - [Prerequisites](#prerequisites)
  - [Repository Structure](#repository-structure)
  - [Configuration](#configuration)
    - [otel-collector-config.yaml](#otel-collector-configyaml)
    - [docker-compose.yml](#docker-composeyml)
  - [Usage](#usage)
    - [1. Clone the Repository](#1-clone-the-repository)
    - [2. Edit Configuration](#2-edit-configuration)
    - [3. Start the Collector](#3-start-the-collector)
    - [4. Verify Operation](#4-verify-operation)
  - [Customizing Receivers \& Exporters](#customizing-receivers--exporters)
  - [Ports and Endpoints](#ports-and-endpoints)
  - [Firewall Requirements](#firewall-requirements)
  - [Troubleshooting](#troubleshooting)
  - [License](#license)

## Overview

The Telemetry Gateway is an OpenTelemetry Collector configured to:

1. **Receive Metrics** via:

   * Prometheus scrape on port 9090
2. **Receive Traces** via:

   * OTLP/gRPC on port 4317
   * OTLP/HTTP on port 4318
   * Jaeger gRPC on port 14250
   * Jaeger HTTP (Thrift) on port 14268
   * Zipkin HTTP on port 9411
3. **Receive Logs** via:

   * OTLP/HTTP (logs over OTLP protocol)

After receiving telemetry data, it **exports** to:

* **Mimir (Prometheus Remote\_Write)** at `http://<MIMIR_HOST>:9009/api/v1/push`
* **Tempo (OTLP/gRPC)** at `<TEMPO_HOST>:4317`
* **Loki (HTTP push)** at `http://<LOKI_HOST>:3100/loki/api/v1/push`

All hosts and telemetry agents within a firewalled compartment can point to this single collector endpoint. The collector, in turn, handles outbound connections to the central LGTM stack.

## Architecture

```plaintext
+-----------------------------+               +-------------------------+
|  Secure Compartment (VPC)   |               | Central LGTM Stack      |
|                             |               | (Loki, Mimir, Tempo)    |
|  +-----------------------+  |               |                         |
|  |  telemetry-gateway    |  |               |  Loki   @ <LOKI_HOST>:3100 <--+
|  |  (OTel Collector)     |  | -- outbound -->  Mimir  @ <MIMIR_HOST>:9009 <--+
|  |  9090, 4317, 4318,    |  |               |  Tempo  @ <TEMPO_HOST>:4317 <--+  
|  |  14250, 14268, 9411   |  |               |                         |  
|  +-----------+-----------+  |               +-------------------------+  
|              |              |                                              
|  +-----------v-----------+  |                                              
|  |  App Host A           |  |                                              
|  |  (Prometheus Exporter)|  |                                              
|  +-----------------------+  |                                              
|                             |                                              
|  +-----------------------+  |                                              
|  |  App Host B           |  |                                              
|  |  (OTEL SDK, Jaeger)   |  |                                              
|  +-----------------------+  |                                              
|                             |                                              
|  +-----------------------+  |                                              
|  |  App Host C           |  |                                              
|  |  (Promtail, Fluentd)  |  |                                              
|  +-----------------------+  |                                              
+-----------------------------+                                              
```

## Prerequisites

* Docker and Docker Compose (v2+) installed on the host executing the collector.
* Network connectivity from this host to the central LGTM endpoints:

  * `<LOKI_HOST>:3100`
  * `<MIMIR_HOST>:9009`
  * `<TEMPO_HOST>:4317`
* Replace `HOST_A`, `HOST_B`, `HOST_C`, `<LOKI_HOST>`, `<MIMIR_HOST>`, and `<TEMPO_HOST>` with actual IPs or DNS names.

## Repository Structure

```plaintext
.
├── docker-compose.yml
├── otel-collector-config.yaml
└── README.md
```

* **docker-compose.yml**: Defines the telemetry-gateway service and exposes ports for receivers.
* **otel-collector-config.yaml**: The OpenTelemetry Collector configuration (receivers, processors, and exporters).
* **README.md**: This documentation.

## Configuration

### otel-collector-config.yaml

This file configures:

* **Receivers**:

  * Prometheus (scrapes local exporters)
  * OTLP (gRPC & HTTP)
  * Jaeger (gRPC & HTTP)
  * Zipkin (HTTP)
* **Processors**:

  * `batch` (buffers and batches telemetry)
* **Exporters**:

  * Prometheus Remote\_Write (to Mimir)
  * OTLP/Tempo (to Tempo)
  * Loki (to Loki)

Make sure to replace the placeholders:

* `<MIMIR_HOST>`: IP or hostname of the Mimir remote\_write endpoint
* `<TEMPO_HOST>`: IP or hostname of the Tempo gRPC endpoint
* `<LOKI_HOST>`: IP or hostname of the Loki HTTP push endpoint

Example excerpt:

```yaml
exporters:
  prometheusremotewrite:
    endpoint: "http://<MIMIR_HOST>:9009/api/v1/push"
    tls:
      insecure: true

  otlp/tempo:
    endpoint: "<TEMPO_HOST>:4317"
    tls:
      insecure: true

  loki:
    endpoint: "http://<LOKI_HOST>:3100/loki/api/v1/push"
```

### docker-compose.yml

Defines a single service `telemetry-gateway`:

* **Image**: `otel/opentelemetry-collector-contrib:latest`
* **Ports**:

  * `9090:9090`  (Prometheus scrape)
  * `4317:4317`  (OTLP gRPC)
  * `9411:9411`  (Zipkin HTTP)
  * `14250:14250` (Jaeger gRPC)
  * `3100:3100`  (Loki push)
* **Volume**:

  * Mounts `otel-collector-config.yaml` into `/etc/otel-collector-config.yaml`

```yaml
version: '3.8'

services:
  telemetry-gateway:
    image: otel/opentelemetry-collector-contrib:latest
    container_name: telemetry-gateway
    restart: unless-stopped
    command: ["--config=/etc/otel-collector-config.yaml"]
    ports:
      - "9090:9090"
      - "4317:4317"
      - "9411:9411"
      - "14250:14250"
      - "3100:3100"
    volumes:
      - ./otel-collector-config.yaml:/etc/otel-collector-config.yaml:ro
```

## Usage

### 1. Clone the Repository

```bash
git clone <repository_url>
cd <repository_directory>
```

### 2. Edit Configuration

1. **Open `otel-collector-config.yaml`** and replace:

   * `HOST_A`, `HOST_B`, `HOST_C` under `receivers.prometheus.config.scrape_configs` with actual target addresses.
   * `<MIMIR_HOST>`, `<TEMPO_HOST>`, `<LOKI_HOST>` under `exporters` with your central Grafana LGTM endpoints.

2. **Verify firewall rules** allow outbound traffic from this host to:

   * `<LOKI_HOST>:3100` (Loki)
   * `<MIMIR_HOST>:9009` (Mimir)
   * `<TEMPO_HOST>:4317` (Tempo)

### 3. Start the Collector

```bash
docker-compose up -d
```

* The container `telemetry-gateway` will start and listen on ports 9090, 4317, 9411, 14250, and 3100.

### 4. Verify Operation

1. **Check container logs**:

   ```bash
   ```

docker logs -f telemetry-gateway

````
   - Look for messages indicating receivers are listening and exporters are initialized.

2. **Test Receivers**:
   - **Prometheus**:
     ```bash
curl -sSf http://localhost:9090/metrics
````

```
 Should return collector metrics.
```

* **Zipkin**:

  ```bash
  ```

curl -sSf [http://localhost:9411/health](http://localhost:9411/health)

````
     Should return a `200 OK` (Zipkin HTTP endpoint).

   - **OTLP/gRPC** (requires grpcurl or OpenTelemetry client):
     ```bash
grpcurl -plaintext localhost:4317 list
````

```
 Should list available gRPC services.
```

3\. **Check Exporters**:

* **Prometheus Remote\_Write**: Verify Mimir’s `/metrics` endpoint shows incoming data or check Mimir logs.
* **Loki**: In Grafana → Explore → Loki, run a query like `{}` to see if any logs arrive.
* **Tempo**: In Grafana → Explore → Traces, query for known span names to confirm traces are arriving.

## Customizing Receivers & Exporters

* **Add/Remove Receivers**:

  * To support additional protocols (e.g., Fluentd OTLP logs), modify the `receivers` block.
  * Each receiver must have a matching entry under `service.pipelines`.

* **Add/Remove Exporters**:

  * To forward data to additional destinations (e.g., an on-prem Prometheus), add a new exporter under `exporters` and reference it in the appropriate pipeline.

* **Processors**:

  * The `batch` processor is included to buffer and batch data. You may add filters, resource annotations, or other processors as needed.

* **TLS & Authentication**:

  * If your central LGTM endpoints require TLS with custom certificates, replace `insecure: true` with proper `ca_file`, `cert_file`, and `key_file` entries under each exporter’s `tls` block.

## Ports and Endpoints

| Port  | Protocol      | Purpose                             |
| ----- | ------------- | ----------------------------------- |
| 9090  | HTTP          | Prometheus receiver (`/metrics`)    |
| 4317  | gRPC (OTLP)   | OTLP/gRPC receiver                  |
| 4318  | HTTP (OTLP)   | OTLP/HTTP receiver                  |
| 14250 | gRPC (Jaeger) | Jaeger gRPC receiver                |
| 14268 | HTTP (Jaeger) | Jaeger HTTP (Thrift) receiver       |
| 9411  | HTTP (Zipkin) | Zipkin HTTP receiver                |
| 3100  | HTTP (Loki)   | Loki push receiver (logs over HTTP) |

> Note: Only ports explicitly exposed in `docker-compose.yml` (9090, 4317, 9411, 14250, 3100) are required. OTLP/HTTP on 4318 and Jaeger HTTP on 14268 are implicitly supported by the `otel/opentelemetry-collector-contrib` image but not published unless you add them.

## Firewall Requirements

Ensure outbound connectivity from this collector host (or container environment) to:

* **Loki**: `<LOKI_HOST>:3100` (TCP/HTTP)
* **Mimir**: `<MIMIR_HOST>:9009` (TCP/HTTP)
* **Tempo**: `<TEMPO_HOST>:4317` (TCP/gRPC)

Within a secure compartment, all other hosts should point to this collector’s IP or hostname (e.g., `http://telemetry-gateway.local:9090`) for metrics, logs, and traces.

## Troubleshooting

* **Configuration Errors**:

  * Run:

    ```bash
    ```

otelcontribcol --config=otel-collector-config.yaml --dry-run

````
    to validate YAML syntax and schema before starting.

- **Connection Refused / Timeout**:
  - Verify the central LGTM endpoints are reachable:
    ```bash
nc -vz <LOKI_HOST> 3100
nc -vz <MIMIR_HOST> 9009
nc -vz <TEMPO_HOST> 4317
````

* Check firewall or security group rules.

* **No Data in Grafana**:

  * Ensure data is flowing into Loki/Mimir/Tempo by querying their health or metrics endpoints:

    ```bash
    ```

curl -sSf http\://\<LOKI\_HOST>:3100/ready
curl -sSf http\://\<MIMIR\_HOST>:9009/metrics
curl -sSf http\://\<TEMPO\_HOST>:4317/health

````
  - Inspect collector logs:
    ```bash
docker logs telemetry-gateway --tail=50
````

## License

This project is provided under the MIT License. See [LICENSE](LICENSE) for details.
