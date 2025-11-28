☁️ Flask & MongoDB on Kubernetes

<div align="center">
<p>
<strong>A cloud-native demonstration of deploying a stateful Python application 



 with autoscaling, persistent storage, and secure authentication.</strong>
</p>
</div>

📖 Overview

This project orchestrates a Python Flask web application connected to a MongoDB database within a Kubernetes cluster. It serves as a proof-of-concept for modern DevOps practices, demonstrating:

Containerization of Python web applications.

StatefulSet management for database persistence.

Horizontal Autoscaling (HPA) based on CPU metrics.

Secret Management for secure database authentication.

Service Discovery via internal DNS.

🛠️ Architecture & Prerequisites

The solution is designed to run on a local Kubernetes cluster (Minikube).

Tech Stack

Application: Flask (Python 3.9)

Database: MongoDB (v5.0+)

Orchestration: Kubernetes (Minikube)

Containerization: Docker

Prerequisites

Ensure the following are installed before deployment:

docker (v20.10+)

minikube (v1.26+)

kubectl (v1.24+)

🚀 Deployment Instructions

Follow these steps to deploy the entire stack from scratch.

1. Start the Cluster

Initialize Minikube and enable the metrics server (required for Autoscaling).

minikube start --driver=docker
minikube addons enable metrics-server


2. Build & Load Images

Since we are using a local cluster, we build the image and load it directly into Minikube's Docker daemon.

# Build the image locally
docker build -t flask-k8s-app:v1 .

# Load the image into Minikube (avoids need for Docker Hub)
minikube image load flask-k8s-app:v1


3. Apply Kubernetes Manifests

Deploy the Secrets, Volumes, Database, and Application.

kubectl apply -f k8s/


4. Verify & Access

Wait for all pods to enter the Running state:

kubectl get pods -w


Once running, retrieve the URL to access the application:

minikube service flask-service --url


🧠 Technical Deep Dive

1. DNS Resolution in Kubernetes

Context: The Flask application connects to MongoDB using the hostname mongodb-service.

Explanation:
Kubernetes maintains an internal DNS service (CoreDNS). When we created the Service named mongodb-service, Kubernetes assigned it the DNS record mongodb-service.default.svc.cluster.local.

When the Flask pod tries to connect to mongodb://admin:password@mongodb-service, it queries the internal DNS.

The DNS resolves this hostname to the ClusterIP of the MongoDB service (or the Pod IP in headless services).

This decouples the application from specific IP addresses, allowing dynamic scaling and healing.

2. Resource Management (Requests vs. Limits)

We configured specific resource constraints to ensure cluster stability.

Resource

Request (Guaranteed)

Limit (Hard Cap)

Purpose

CPU

0.2 (200m)

0.5 (500m)

Requests help the scheduler find a node with enough capacity. Limits prevent a single pod from starving the node of CPU cycles.

Memory

250Mi

500Mi

Memory limits prevent memory leaks from crashing the entire node (OOMKilled).

💡 Design Choices

Why StatefulSet for MongoDB?

Decision: Deployed MongoDB as a StatefulSet rather than a Deployment.
Reasoning:

Stable Network Identity: StatefulSets provide predictable names (mongodb-0), which are essential for database clusters and replication.

Ordered Deployment: Ensures the database starts up in a specific order.

Persistent Storage: If the pod crashes and restarts on a different node, the PersistentVolumeClaim (PVC) automatically re-attaches to the new pod, ensuring zero data loss.

Security Implementation

Secrets: Database credentials (MONGO_INITDB_ROOT_USERNAME, PASSWORD) are stored in a Kubernetes Secret object (base64 encoded) and injected as environment variables. This avoids hardcoding passwords in the YAML files.

🧪 Testing Scenarios & Results

1. Database Persistence Test

We verified that data survives pod restarts.

Action: Sent a POST request to /data to insert a record.

Action: Deleted the MongoDB pod (kubectl delete pod mongodb-0).

Result: After the pod auto-restarted, a GET request successfully retrieved the previously saved data.

2. Autoscaling (HPA) Stress Test

We simulated a traffic spike to test the Horizontal Pod Autoscaler.

Configuration: Scale replicas if CPU utilization > 70%.

Simulation Command:

while true; do curl -X POST -H "Content-Type: application/json" -d '{"loadtest": "true"}' http://<MINIKUBE-URL>/data; done


Observation:

Baseline: 2 Replicas (< 1% CPU).

Load Applied: CPU spiked to 90%.

Result: HPA scaled deployment to 3 Replicas automatically.

📸 Proof of Autoscaling

(Screenshot showing Replicas increasing from 2 to 3 under load)

📂 Project Structure

flask-mongodb-app/
├── app.py                  # Flask Application Source
├── requirements.txt        # Python Dependencies
├── Dockerfile              # Container Definition
├── k8s/                    # Kubernetes Manifests
│   ├── flask-deployment.yaml
│   ├── flask-service.yaml
│   ├── mongo-statefulset.yaml
│   └── mongo-config.yaml
└── README.md               # Documentation
