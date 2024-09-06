# SRE day 2024 - London

## Debezium Server Deployment with Debezium Operator on Kubernetes

**This demonstration showcases how to deploy Debezium Server using the Debezium Operator on Kubernetes to enable streaming data from a PostgreSQL database into Redis. The demo is designed to be a simple showcase and is not intended for production-ready deployment.**

### Overview

This demo demonstrates an easy way to stream data changes from a PostgreSQL database to Redis using Debezium Server. Debezium captures database changes using Change Data Capture (CDC) and streams them into Redis, making it ideal for real-time data processing.

### Prerequisites

- Kubernetes cluster (Minikube, Kind, or any other Kubernetes setup)
- kubectl CLI
- Helm CLI
- Docker (optional, for local development)

### Architecture

The architecture of the demonstration consists of the following components:

- PostgreSQL Database: Source database where changes are captured.
- Debezium Operator: Manages Debezium Server deployment
- Debezium Server: Listens to changes from PostgreSQL and streams them to Redis.
- Redis: Target data store that receives changes streamed by Debezium.
- [DMT](https://github.com/skodjob/database-performance-hub): Database manipulation tool -- Allows you to manipulate database and redis just via HTTP API

### Setup Guide
1. Clone the Repository

Clone the repository containing the necessary Kubernetes manifests and configuration files:


```bash
git clone https://github.com/obabec/sreday-2024-demo
cd sreday-2024-demo
```
2. Deploy Kubernetes Cluster

Ensure your Kubernetes cluster is up and running. If using Kind, you can start it with:

```bash
kind create cluster
```
3. Create namespace/project
```bash
kubectl create namespace demo
```

4. Deploy PostgreSQL Database

Deploy PostgreSQL using the provided YAML manifest:

```bash
kubectl apply -f postgres.yaml -n demo
```

Verify the deployment:

```bash
kubectl get pods -l app=postgresql -n demo
```

5. Deploy Redis

Deploy Redis to serve as the sink for the streamed data:

```bash

kubectl apply -f redis.yaml -n demo
```
Verify the deployment:

```bash
kubectl get pods -l app=redis
```

6. Deploy Debezium Operator

Deploy the Debezium Operator to manage Debezium Server deployments:

```bash
helm repo add  --force-update debezium https://charts.debezium.io
helm install debezium-operator debezium/debezium-operator --version 2.7.0-beta2 -n demo
```

Wait for operator to start

```bash
kubectl get pods -l name=debezium-operator
```

7. Deploy Debezium Server

Deploy the Debezium Server configured to capture changes from PostgreSQL and stream them into Redis:

```bash
kubectl apply -f server.yaml
```

Check if Debezium Server is running:

```bash
kubectl get pods -l app=debezium-server
```

8. Deploy DMT
```bash
kubectl apply -f dmt -n demo
```
Port forward DMT pod

```bash
kubectl port-forward -n demo service/database-manipulation-tool 8080:80
```
### Start playing
Once you have everything deployed you can start pushing changes to table `public.outside_temp` and inspect the events in `sredays.public.outside_temp` Redis streams.

Example requests to upsert data to table:
**Target**

`http://localhost:8080/Main/CreateTableAndUpsert`


**Payload**
```json
{
    "name": "outside_temp",
    "primary": "device_id",
    "columnEntries": [
        {
            "columnName": "device_id",
            "dataType": "Integer",
            "value": "2"
        },
        {
            "columnName": "current_temp",
            "dataType": "Double",
            "value": "23.2"
        }
    ]
}
```
Read Redis Stream:

**Target**
`http://localhost:8080/Redis/pollMessages?max=8`

**Payload**
```json
[
    "sredays.public.outside_temp"
]
```

If you want to play with DMT little bit more, check the project [README](https://github.com/skodjob/database-performance-hub/tree/main/database-manipulation-tool).

### Cleaning Up

To clean up the demo, run the following commands:

```bash

kubectl delete namespace demo
```