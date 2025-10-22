---
title: Getting Started
weight: 2
type: docs
---

This document describes how to start RCABench quickly·

## Prerequisites

### Hardware Requirements

| Component   | Requirement                        |
| ----------- | ---------------------------------- |
| **CPU**     | 4+ cores                           |
| **Memory**  | 8GB+ RAM                           |
| **Storage** | 20GB+ available SSD storage        |
| **OS**      | Linux (Ubuntu 20.04+ or CentOS 8+) |

### Software Requirements

- **[Docker](https://docs.docker.com/engine/install/)** (>= 20.10) for local deployment
- **[Kubernetes](https://kubernetes.io/docs/tasks/tools/)** (>= 1.25) or **kind/minikube** for production deployment
- **[Kubectl](https://kubernetes.io/docs/tasks/tools/)** (compatible with your cluster version)
- **[Helm](https://helm.sh/docs/intro/install/)** (>= 3.0) for Kubernetes deployments
- **[Devbox](https://www.jetify.com/docs/devbox/installing-devbox)** for managing development environment
- **[Skaffold](https://skaffold.dev/docs/install/)** for building and deploying services
- **Make** for using the provided Makefile
- **Go** (>= 1.23) for development
- **Python** (>= 3.10) for SDK usage

## Quick Start

{{% steps %}}

### Install Chaos Mesh

{{< cards cols="1" >}}
{{< card link="../deployment/chaos-mesh" title="Install Chaos Mesh" icon="arrow-circle-right" >}}
{{< /cards >}}

### Deploy Train Ticket

{{< cards cols="1" >}}
{{< card link="../deployment/pedestals/train-ticket" title="Deploy Train Ticket" icon="arrow-circle-right" >}}
{{< /cards >}}

### Local Development Setup

{{< cards cols="1" >}}
{{< card link="../deployment/rcabench" title="Deploy RCABench" icon="arrow-circle-right" >}}
{{< /cards >}}

{{% /steps %}}
