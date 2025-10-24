---
title: Train Ticket
weight: 1
type: docs
---

This document describes how to deploy the `train-ticket` pedestal microservices application for fault injection and root cause analysis benchmarking.

{{< callout type="info" icon="github">}}
**Source Code:** [LGU-SE-Internal/train-ticket](https://github.com/LGU-SE-Internal/train-ticket)
{{< /callout >}}

## Prerequisites

Before you begin, ensure you have the following tools installed:

1. **[Helm](https://helm.sh/docs/intro/install/)**: For managing Kubernetes applications.
2. **[Docker](https://docs.docker.com/engine/install/)**: For building container images.
3. **JDK and Maven**: For building the Java-based project. (JDK 8 or 17 is recommended; ensure your `JAVA_HOME` is set correctly.)
   - You can check your Java version with `java -version`
   - Maven can be installed via package managers or downloaded from the [Apache Maven website](https://maven.apache.org/download.cgi).

> [!Important]
> You need access to a container registry (e.g., Docker Hub, a private registry) to push and pull images.

## Installation

```bash
# Add the Train Ticket Helm repository
helm repo add train-ticket https://lgu-se-internal.github.io/train-ticket
helm repo update
helm search repo train-ticket
```

### Automated Deployment

Use the **RCABench Makefile** to deploy the `train-ticket`.

```bash
# PEDESTAL_KEY is the prefix for the pedestal namespace (e.g., ts for train-ticket)
# PEDESTAL_COUNT is the number of pedestal instances to deploy
make install-releases PEDESTAL_KEY=ts PEDESTAL_COUNT=1
```

### Manual Deployment

{{< tabs items="CN Mirror, Global Mirror" >}}

{{< tab >}}

```bash
helm install ts0 train-ticket/trainticket -n ts0 --create-namespace \
	--set global.image.repository=pair-diagnose-cn-guangzhou.cr.volces.com/opspai \
	--set global.image.tag=v1.0.0-213-gf9294111 \
	--set global.security.allowInsecureImages=true \
	--set mysql.image.repository=pair-diagnose-cn-guangzhou.cr.volces.com/library/mysql \
  	--set rabbitmq.image.registry=pair-diagnose-cn-guangzhou.cr.volces.com \
  	--set rabbitmq.image.repository=bitnamilegacy/rabbitmq \
  	--set loadgenerator.image.repository=pair-diagnose-cn-guangzhou.cr.volces.com/opspai/loadgenerator \
  	--set loadgenerator.initContainer.image=pair-diagnose-cn-guangzhou.cr.volces.com/nicolaka/netshoot:v0.14 \
	--set services.tsUiDashboard.nodePort=31000
```

{{< /tab >}}

{{< tab >}}

```bash
helm install ts0 train-ticket/trainticket -n ts0 --create-namespace \
    --set global.image.tag=v1.0.0-213-gf9294111 \
	--set global.security.allowInsecureImages=true \
	--set services.tsUiDashboard.nodePort=30081
```

{{< /tab >}}

{{< /tabs >}}

## Verification

```bash
# Check all pods are running
kubectl get pods -n ts0
```
