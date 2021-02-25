# local-monitoring-stack
## Overview
This project was built in order to easily spin up a handful of open-source monitoring solutions in containers for development.  In addition, it has been built toward exporting OpenTelemetry Spans to AppDynamics, so there are components setup to do so, and require some futher configuration to make them work.

The stack consists of:
- Prometheus
- Jaeger
- Loki
- Grafana
- cAdvisor
- OpenTelemetry Collector (no agent)

More information about the [Compose Services](#docker-compose-services) below.  For the most part, I am just using the latest images available.

   > __Note:__  This project was built/tested only on Docker for Mac, and uses Docker-Compose

It's not necessary to build anything in this project.  All images can be pulled from Docker Hub when you run with [Docker Compose](#quick-start-with-docker-compose).

## Quick Start with Docker Compose
### Prerequisites
In order to run this project, you'll need:
- Docker for Mac
- Docker Compose 
  <br />  

   > __Note:__  The Docker versions must support Docker Compose File version 3.2+

### Steps to Run
1. Clone this repository to your local machine.
2. Copy the `.env_public` file to a file named `.env` in the root project directory, and configure it appropriately.

   > __IMPORTANT:__ Detailed information regarding `.env` file can be found [below](#env-file).  This __MUST__ be done for this project to work!

3. Copy the `otel-collector-config.yaml` file to a file named `appd-otel-collector-config.yaml` in the root project directory, and configure it appropriately.

   > __IMPORTANT:__ Detailed information regarding `appd-otel-collector-config.yaml` file can be found [below](#appd-otel-collector-config.yaml-file).  This __MUST__ be done for this project to work!

4. Create external docker network named `monitoring`.
```bash
$ docker network create monitoring
```
5. Use Docker Compose to start
```bash
$ docker-compose up -d
```

## Docker Compose Services
### prometheus
Prometheus metrics backend.  

Prometheus Image, pulled from [`prom/prometheus:latest`](https://hub.docker.com/r/prom/prometheus)

By default, accessible on `http://$DOCKER_HOSTNAME:9091`.

### jaeger
Jaeger tracing backend.  

Jaeger "all-in-one" Image, pulled from [`jaegertracing/all-in-one:latest`](https://hub.docker.com/r/jaegertracing/all-in-one).  

By default, accessible on `http://$DOCKER_HOSTNAME:16686`.

### loki
Loki logging backend.

Loki Image, pulled from [`grafana/loki:1.6.0`](https://hub.docker.com/r/grafana/loki).  

There is no UI, Loki is a datasource for Grafana. 

### grafana
Grafana Server.

Grafana Community Image, pulled from [`grafana/grafana:latest`](https://hub.docker.com/r/grafana/grafana).  

Accessible on `http://$DOCKER_HOSTNAME:3000`

Login Credentials: `admin/admin`

Grafana has been setup to automatically configure the Loki and Prometheus Data Sources.

### cAdvisor
cAdvisor Agent.

Grafana Community Image, pulled from [`gcr.io/cadvisor/cadvisor:latest`](https://hub.docker.com/r/google/cadvisor/).  

Accessible on `http://$DOCKER_HOSTNAME:8085`

### otel-collector
OpenTelemetry Collector.

OpenTelemetry development Image, pulled from [`otel/opentelemetry-collector-dev:latest`](https://hub.docker.com/r/otel/opentelemetry-collector-dev).  

There is no UI, merely a pipeline that is configured via `appd-otel-collector-config.yaml`.


## More Notes on Configuration

### docker-compose.yml
This file is located in the project root and manages building and running the Docker containers. It uses the `.env` file to populate environment variables for the project to work properly.

### .env File
This file contains all of the environment variables that need to be populated in order for the project to run, and for the performance tools to operate.  Items that *must* be tailored to your environment are:

#### AppDynamics Controller Configuration - N/A
This configuration is used by the AppD Agent (in this case, the Machine Agent) to connect to the AppD Controller of your choice - currently the AppD Machine Agent is disabled.
```bash
# AppD Agent
APPDYNAMICS_AGENT_ACCOUNT_ACCESS_KEY=<access_key>
APPDYNAMICS_AGENT_ACCOUNT_NAME=<customer1>
APPDYNAMICS_CONTROLLER_HOST_NAME=<controller_host>
APPDYNAMICS_CONTROLLER_PORT=<8090>
APPDYNAMICS_CONTROLLER_SSL_ENABLED=<false_or_true>
APPDYNAMICS_AGENT_APPLICATION_NAME=GarageSale-Compose
# Tier and Node names are set in docker-compose.yml individually
```
> __Tip:__  Documentation on these configuration properties can be found in the [AppDynamics Machine Agent Configuration Properties](https://docs.appdynamics.com/display/PRO20X/Machine+Agent+Configuration+Properties)

#### AppDynamics Analytics Agent Configuration - N/A
See above, AppD Machine Agent is disabled.
```bash
# Analytics Config
APPDYNAMICS_AGENT_GLOBAL_ACCOUNT_NAME=<global_account_name>
EVENT_ENDPOINT=<http://192.168.86.40:9080>
```  

#### AppDynamics Machine Agent Server Visibility and Docker Visibility Configuration - N/A
See above, AppD Machine Agent is disabled.
```bash
# AppD Machine Agent/Docker Visibility
APPDYNAMICS_SIM_ENABLED=<true_or_false>
APPDYNAMICS_DOCKER_ENABLED=<true_or_false>
APPDYNAMICS_AGENT_ENABLE_CONTAINERIDASHOSTID=<true_or_false>
APPDYNAMICS_AGENT_UNIQUE_HOST_ID=DockerForMacOS
```  

#### AppDynamics OpenTelemetry Ingestion Service API Key
API Key for the AppDynamics OpenTelemetry Ingestion Service.
```bash
# AppD OTel Ingest API Key
X_API_KEY=<my_x_api_key>
```

### appd-otel-collector-config.yaml File

#### OpenTelemetry Processors
The `value:` sections of the below should be configured appropriately.
```bash
processors:
  resource:
    attributes:
      - key: appdynamics.controller.account
        action: upsert
        value: "my-appd-account"
      - key: appdynamics.controller.host
        action: upsert
        value: "my-controller-host"
      - key: appdynamics.controller.port
        action: upsert
        value: 443
  batch:
  ```

#### OpenTelemetry Exporters
The `endpoint:` section of the below should be configured appropriately.  Note, the `x-api-key` header is defined in the `.env` file.
  ```bash
exporters:
  ...
  otlphttp:
    endpoint: https://my-appd-endpoint.com
    headers: {"x-api-key": "${X_API_KEY}"}
```
