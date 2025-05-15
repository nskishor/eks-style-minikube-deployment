# EKS-Style Secure Microservices Deployment on Minikube

## Project Overview

This project demonstrates how to build and secure a Kubernetes microservices environment on Minikube that mimics AWS EKS production patterns. It includes deploying multiple services, simulating IAM with MinIO, applying network policies for security, monitoring with Prometheus and Grafana, and handling a simulated security incident involving sensitive data leakage.


## Setup Instructions

## Prerequisites

- Minikube  
- Docker  
- kubectl  
- Helm  

**Step 1: Start Minikube:**

minikube start --cpus=2 --memory=4096 --addons=ingress,metrics-server


**Step 2: Deploy Microservices with Helm:**

Deploy the three microservices into the app-namespace:

helm install gateway ./helm-charts/gateway -n app-namespace --create-namespace
helm install auth-service ./helm-charts/auth-service -n app-namespace
helm install data-service ./helm-charts/data-service -n app-namespace

**Step 3: Deploy MinIO for IAM Simulation:**
Apply the Kubernetes manifests to deploy MinIO and related resources:

kubectl apply -f manifests/namespaces.yaml
kubectl apply -f manifests/secrets.yaml
kubectl apply -f manifests/serviceaccounts.yaml
kubectl apply -f manifests/minio-deployment.yaml

**Step 4: Apply Network Policies:**
Restrict network traffic to enforce least privilege communication:

kubectl apply -f manifests/networkpolicies.yaml

**Step 5: Setup Monitoring with Prometheus & Grafana:**
Add Prometheus and Grafana monitoring stack:

helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install monitoring prometheus-community/kube-prometheus-stack -n monitoring --create-namespace

Import the Grafana dashboard located at grafana/dashboard.json into Grafana UI for monitoring metrics like CPU, memory, HTTP request rates, and pod restarts.

**Architecture Diagram:**
see architecture.txt for an ASCII version.

**Design Decisions:**
Helm Charts: Chosen for modularity and ease of deployment for each microservice.
Namespace Separation: app-namespace for application services, separate namespaces for system components to isolate resources.
Probes: Liveness and readiness probes added to ensure healthy pod lifecycle management.
MinIO: Used as a local mock S3 for simulating IAM policies via Kubernetes ServiceAccounts.
Network Policies: Enforce strict communication rules based on least privilege principles.
Observability: Prometheus and Grafana provide real-time monitoring and alerting capabilities.
Security Policies: Kyverno policies detect and block any unauthorized logging of sensitive Authorization headers.

**Security Incident & Fix:**
Incident:
The auth-service was leaking sensitive Authorization headers in its logs and had unintended access to external services.

**Investigation:**
Using kubectl logs and k9s, I identified the leak of Authorization headers and unauthorized egress traffic.

**Resolution:**
Updated the auth-service deployment to sanitize logs and avoid logging Authorization headers.
Applied NetworkPolicy to restrict auth-service communication only to data-service.
Created Kyverno policies to detect and block pods logging Authorization headers.
Verified policies by simulating header logging attempts and ensuring they were blocked or alerted.

**Assumptions & Known Issues:**
This setup is designed for local Minikube and does not connect to real AWS services.
Ingress is exposed via Minikube’s default ingress addon, suited for local development only.
IAM simulation is simplified using MinIO and Kubernetes RBAC, not full AWS IAM.
Security policies are basic and intended as examples, extensible for production use.
Pod failure recovery tested manually by deleting pods; no automated failure injection tools used.

**Failure Simulation:**
You can simulate a pod failure to test system resiliency:
kubectl delete pod <pod-name> -n app-namespace
Watch Kubernetes recreate the pod and observe related metrics and alerts in Grafana.
