---
layout: post
title: "Microservice health check in Kubernetes"
date: 2018-01-23 16:31
---
# TL;DR
Service should provide a standard endpoint for the purpose of health check and monitoring. The specification for the endpoint should conform to the requirements as elaborated in section [Requirements](#Requirements).

# Background
## what is health check
A health check detects the healthy status of a service, reporting whether the service is able to handle requests or whether the service is in a bad state and should be restarted.

## Why health check is needed

### High availability
There are many cases when a service is started/restarted
- instance/pod restart
- service/deployment up scaling
- rolling update

Under these circumstances, if a request is forwared to a service that is still in the middle of its starting/restarting process, it would probably fail. So we need to make sure a service is healthy to accept requests before adding it to the load balancer(kubernetes service), such that we could reduce the service down time and achieve high availability.

### Service stability
Service running for a long period of time may fall into a bad state, in which service is unable to handle requests properly. In this case, service needs to be prohibited from receiving requests, until it is recovered either via restart or manual resurrection. Thus our service in all is stable.

### Monitoring
A big part of the DevOps responsibilities is to monitor and maintain the health of running services. If a service goes down, appropriate actions should be undertaken to bring the service back to life. Health check informs the DevOps whether the service is malfunctioning.

## Clients of health checks
- Load balancer (Kubernetes service)
- Monitoring service (Prometheus probe)
- Pods (Readiness/Liveness probe)

## Downsides of health check
As health check is done periodically, not in a real time manner, there still could be time gap before the unhealthy state is known to the clients. To mitigate the effect of this situation, a reasonable checking period should be set.

# Requirements
## What should be checked
As the definition of healthy may vary from service to service, depending on the service application logics, there could be many levels of healthy:
- the service is up
- the service is up and the infrastructure service used by the service is healthy
- the service is up, the infrastructure service used by the service is healthy, the dependent microservice is healthy
- the service is up, the infrastructure service used by the service is healthy, the dependent microservice is healthy, smoke tests are passed

Each service may define its own criteria, however the result of these checks should be certain, ie, the service is either healthy or not healthy, there should be no middle state.

## How to expose health check to clients
- The service should implement the health check in a RESTful API manner.
- The endpoint is unified as "/health"

## How health check respond to clients
### Status code
- 200 OK for healthy
- 503 Service Unavailable for unhealthy

### Response body
Response body can be empty, however attaching additional information of what is checked and the result of the check is preferred

## Security/Access control
The health check should be private and limited to internal access, however if it is open to public access:
- For unauthenticated access, service should provide a basic health info, returning a UP/DOWN status
- For authenticated access, service may provide more detail health info

# Implementation
## Examples

**Service OK**

```
$ curl -XGET http://127.0.0.1:9000/health
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
 
{
    "status": "UP"
}
```

**Service Unavailable**

```
$ curl -XGET http://127.0.0.1:9000/health
HTTP/1.1 503 Service Unavailable
Content-Type: application/json; charset=utf-8
 
{
    "status": "Down"
}
```

**Authenticated access**

```
$ curl -XGET http://127.0.0.1:9000/health -H 'Authorization: Basic ZnNfbm9ybWFsOkBDZ0JkSjZOKz9TbmQhRytIJEI3'
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
 
{ 
  "status":"UP",
  "fooService":{ 
    "status":"UP",
    "description":"Foo service"
  },
  "mysql":{ 
    "status":"UP",
    "description":"MySQL Database",
    "hello":1
  }
}
```

## Libraries
### Java
[Spring Boot Actuator](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#production-ready-health)

### Go
N/A

# Client Integration

## Kubernetes integration
Please refer to https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/

Readiness and liveness probes can be used in parallel for the same container. Using both can ensure that traffic does not reach a container that is not ready for it, and that containers are restarted when they fail.

### Readiness Probe
```
readinessProbe: # check if service in a healthy state, will remove pod from service/loadbalancer if probe failed
    httpGet:
        path: /health
        port: 9000
    initialDelaySeconds: 10 # start checking after 10s after pod starts. should set to a minimal value such that service able to receive requests as soon as it is ready
    periodSeconds: 10 # check health check api every 10 seconds
    timeoutSeconds: 3 # if response time is logger than 3 seconds, we consider the check as failed
    failureThreshold: 3  # if check fails for 3 times in a row, we consider the pod is in a bad state, pod will be restarted
    successThreshold: 1 # if check succeeds for once, we consider the pod is back to normal
```

### Liveness Probe
```
livenessProbe: # check if pod is in a bad state, will restart pod if probe failed
    httpGet:
        path: /health
        port: 9000
    initialDelaySeconds: 180 # start checking after 180s after pod starts, should be logger than service start time. Some service takes minutes to start, so we set a big value here.
    periodSeconds: 10 # check health check api every 10 seconds
    timeoutSeconds: 3 # if response time is logger than 3 seconds, we consider the check as failed
    failureThreshold: 3 # if check fails for 3 times in a row, we consider the pod is in a bad state, pod will be restarted
    successThreshold: 1 # if check succeeds for once, we consider the pod is back to normal
```

## Prometheus integration

Prometheus keeps polling health API constantly and store the result in its time series database. If health check metrics match a predefined alert rule, a alert will be triggered.

### Scrape config

```
job_name: 'health-check'
  metrics_path: /probe
  params:
    module: [http_2xx]  # Look for a HTTP 200 response.
  kubernetes_sd_configs:
  - role: service
 
  relabel_configs:
    - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_healthcheck]
      regex: true
      action: keep
    - source_labels: [__meta_kubernetes_service_name]
      target_label: service
    - source_labels: [__address__]
      regex: (.*)(:80)?
      target_label: __param_target
      replacement: ${1}/health
    - source_labels: [__param_target]
      regex: (.*)
      target_label: instance
      replacement: ${1}
    - source_labels: []
      regex: .*
      target_label: __address__
      replacement: blackbox-exporter-service:9115  # Blackbox exporter.
```

### Service annotation
Add *prometheus.io/healthcheck* annotation to Kubernetes service so that they could be discovered by the health check job.
```
apiVersion: v1
kind: Service
metadata:
  annotations:
    prometheus.io/healthcheck: "true"
  name: companyprofile-service
  namespace: hp
  labels:
    app: companyprofile-service
spec:
  ports:
  - port: 80
    targetPort: 8087
    protocol: TCP
  selector:
    app: companyprofile
```

### Blackbox exporter config

Config a *http_2xx* module to scrape health api
```
scrape_configs:
- job_name: 'blackbox'
metrics_path: /probe
params:
module: [http_2xx] # Look for a HTTP 200 response.
static_configs:
- targets:
- http://authentication-service.hp/health # Target to probe
relabel_configs:
- source_labels: [__address__]
target_label: __param_target
- source_labels: [__param_target]
target_label: instance
- target_label: __address__
replacement: 127.0.0.1:9115 # Blackbox exporter.
```
