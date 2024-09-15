# RabbitMQ Autoscaling with KEDA and Kubernetes

---

## Overview

This guide will walk you through setting up an autoscaling environment where a RabbitMQ consumer scales based on the queue length using Kubernetes and KEDA.

The steps include:

1. Prerequisites
2. Set Up a Kubernetes Namespace
3. Build and Push the producer/consumer Docker Images
4. Install KEDA
5. Deploy RabbitMQ
6. Deploy the Producer Application
7. Deploy the Consumer Application
8. Configure KEDA for Autoscaling
9. Verify the Autoscaling Setup
10. Clean Up Resources

---

## **1. Prerequisites**

Before you begin, ensure you have the following installed and configured:

- **Kubernetes Cluster**: A running Kubernetes cluster (e.g., Minikube, Kind, Docker Desktop, or a cloud provider's managed Kubernetes service).
  - I usually prefer using minikube. 
  - You can install it using the following command:
    ```bash
    brew install minikube
    ```
    You can start the minikube cluster using the following command:
    ```bash
    minikube start
    ```
- **kubectl**: Command-line tool to interact with your Kubernetes cluster.
- **Docker**: For building and pushing Docker images.
- **Helm**: Kubernetes package manager, used to install KEDA.

### Note

This guide assumes you have administrative access to your Kubernetes cluster and can create namespaces, deploy applications, and install cluster-wide resources like KEDA.

---

## 2. Set Up a Kubernetes Namespace

Creating a dedicated namespace helps isolate your resources and is a good practice for managing Kubernetes environments.

### Create a Namespace

We'll create a namespace called `rabbitmq-autoscaling`.

```bash
kubectl create namespace rabbitmq-autoscaling
```

### Switch to the New Namespace

You can configure your `kubectl` context to use this namespace by default:

```bash
kubectl config set-context --current --namespace=rabbitmq-autoscaling
```

---

## 3. Build and Push Docker Images

Run all these commands from the project root.

1. Build the Producer Image

```bash
docker build -t your-dockerhub-username/producer:latest -f Dockerfile.producer .
```

2. Build the Consumer Image

```bash
docker build -t your-dockerhub-username/consumer:latest -f Dockerfile.consumer .
```

Ensure you're logged in to your Docker registry (e.g., Docker Hub):

```bash
docker login
```

3. Push the Producer Image

```bash
docker push your-dockerhub-username/producer:latest
```

4. Push the Consumer Image

```bash
docker push your-dockerhub-username/consumer:latest
```

---

## 4. Install KEDA

KEDA enables event-driven autoscaling for Kubernetes workloads. We'll install it using Helm.

### Add the KEDA Helm Repository

```bash
helm repo add kedacore https://kedacore.github.io/charts
helm repo update
```

### Install KEDA into the Namespace

While KEDA can be installed cluster-wide, for isolation, we'll install it into our namespace.

```bash
helm install keda kedacore/keda --namespace rabbitmq-autoscaling
```

#### Verify KEDA Installation

```bash
kubectl get pods -n rabbitmq-autoscaling
```

You should see KEDA pods running.

---

## Deploy RabbitMQ

We'll deploy RabbitMQ with the management plugin enabled.

### Create a Secret for RabbitMQ Credentials

```bash
kubectl apply -f k8s/rabbitmq/rabbitmq-secret.yaml
```

### Deploy RabbitMQ

```bash
kubectl apply -f k8s/rabbitmq/rabbitmq-deployment.yaml
```

#### Verify RabbitMQ Deployment

```bash
kubectl get pods -n rabbitmq-autoscaling -l app=rabbitmq
```

---

## Deploy the Producer Application

The producer sends messages to the RabbitMQ queue.

```bash
kubectl apply -f k8s/producer/producer-deployment.yaml
```

### Verify Producer Deployment

```bash
kubectl get pods -n rabbitmq-autoscaling -l app=producer
```

---

## Deploy the Consumer Application

The consumer processes messages from the RabbitMQ queue.

```bash
kubectl apply -f k8s/consumer/consumer-deployment.yaml
```

### Verify Consumer Deployment

```bash
kubectl get pods -n rabbitmq-autoscaling -l app=consumer
```

---

## Configure KEDA for Autoscaling

We'll create a `TriggerAuthentication` and a `ScaledObject` to enable autoscaling based on RabbitMQ queue length.

### Create TriggerAuthentication

```bash
kubectl apply -f k8s/consumer/rabbitmq-trigger-auth.yaml
```

### Create ScaledObject

```bash
kubectl apply -f k8s/consumer/consumer-scaledobject.yaml
```

---

## Verify the Autoscaling Setup

### Monitor the Consumer Deployment

Start watching the consumer deployment to observe scaling behavior.

```bash
kubectl get deployment consumer -n rabbitmq-autoscaling -w
```

### Observe Autoscaling

As the producer sends messages to the `task_queue`, the queue length increases. Once it exceeds the activation value set in the `ScaledObject` (10 messages), KEDA will scale up the consumer deployment.

### Access RabbitMQ Management UI (Optional)

To monitor the queue directly:

1. Port Forward to RabbitMQ Management UI

```bash
kubectl port-forward service/rabbitmq 15672:15672 -n rabbitmq-autoscaling
```

2. Access the UI

Open your web browser and navigate to [http://localhost:15672](http://localhost:15672).

3. Login Credentials

- **Username:** `guest`
- **Password:** `guest`

You can view the `task_queue` and observe the message rates and queue length.

### Check KEDA Metrics (Optional)

You can check the status of the `ScaledObject`:

```bash
kubectl describe scaledobject consumer-scaledobject -n rabbitmq-autoscaling
```

---

## Clean Up Resources

When you're finished, you can delete all the resources and the namespace:

```bash
# Delete resources
kubectl delete -f k8s/ --recursive --namespace rabbitmq-autoscaling

# Uninstall KEDA
helm uninstall keda --namespace rabbitmq-autoscaling

# Delete the namespace
kubectl delete namespace rabbitmq-autoscaling
```

---

## **Troubleshooting**

- **Pods Not Starting:**

  ```bash
  kubectl logs pod-name -n rabbitmq-autoscaling
  ```

- **Consumer Not Scaling:**

    - Check KEDA operator logs:

      ```bash
      kubectl logs deployment/keda-operator -n rabbitmq-autoscaling
      ```

    - Verify the `ScaledObject` status.

- **Connection Issues:**

    - Ensure services can resolve DNS names.
    - Verify network policies or firewalls are not blocking communication.

---
