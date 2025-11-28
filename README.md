Flask MongoDB Kubernetes Deployment (Devops Intern Assignment)

This project contains a Python Flask application connected to a MongoDB database, deployed on a Kubernetes cluster. It demonstrates containerization, stateful application management, secure configuration, and horizontal autoscaling practices as per the assignment requirements.

Prerequisites

Docker: For building and managing container images.

Minikube: For running a local Kubernetes cluster.

Kubectl: For interacting with the Kubernetes cluster.

Build and Push Instructions

1. Build the Docker Image

To package the Flask application into a container image:

docker build -t flask-k8s-app:v1 .


2. Push to Container Registry

Note: For this Minikube deployment, the image is loaded directly into the cluster. If deploying to a remote cluster like EKS/GKE, use these steps:

# Tag the image for your registry (eg, Docker Hub)
docker tag flask-k8s-app:v1 your-dockerhub-username/flask-k8s-app:v1

# Push the image
docker push your-dockerhub-username/flask-k8s-app:v1


3. Load Image Locally

Since Minikube is used locally, the image is loaded directly to avoid pulling from a remote registry:

minikube image load flask-k8s-app:v1


Deployment Steps

Follow these steps to deploy the application and database on Minikube.

Start Minikube & Enable Metrics
The metrics-server is required for the Horizontal Pod Autoscaler (HPA) to function.

minikube start
minikube addons enable metrics-server


Deploy Kubernetes Resources
Apply all configuration files (Secrets, StatefulSet, Deployment, Services, HPA) located in the k8s/ directory.

kubectl apply -f k8s/


Verify Pod Status
Wait until all pods (MongoDB and Flask app) are in the Running state.

kubectl get pods


Access the Application
Retrieve the URL exposed by Minikube to access the Flask web server.

minikube service flask-service --url


Technical Concepts and Explanations

1. DNS Resolution in Kubernetes

Question: How does the Flask app connect to MongoDB without a hardcoded IP?

Explanation: Kubernetes includes an internal DNS server (CoreDNS).

We defined a Service for MongoDB named mongodb-service.

Kubernetes automatically assigns this Service a DNS record: mongodb-service.default.svc.cluster.local.

Inside the cluster, any pod can resolve the hostname mongodb-service to the IP address of the MongoDB pod.

The Flask app uses this hostname in its connection string (mongodb://... @mongodb-service ...), allowing dynamic discovery even if the database pod restarts and gets a new IP.

2. Resource Requests vs. Limits

Question: Why are resource configurations important?

Explanation:

Requests (requests):

Definition: The minimum amount of CPU/RAM the container is guaranteed.

Use Case: The Kubernetes Scheduler uses this value to decide which Node has enough capacity to place the Pod.

Configuration: 0.2 CPU, 250Mi Memory.

Limits (limits):

Definition: The maximum amount of CPU/RAM the container is allowed to consume.

Use Case: Ensures application stability and protects the Node. If a container exceeds the CPU limit, it is throttled. If it exceeds the Memory limit, it is terminated (OOMKilled) to prevent it from starving other system processes.

Configuration: 0.5 CPU, 500Mi Memory.

Design Choices

StatefulSet for MongoDB

Choice: StatefulSet

Reasoning: Unlike stateless web apps, databases require persistent storage and a stable network identity (e.g., mongodb-0). A standard Deployment creates interchangeable pods; if a DB pod restarted in a Deployment, it might lose its identity or data volume mapping. A StatefulSet guarantees ordering and persistence.

Kubernetes Secrets

Choice: Kubernetes Secret object.

Reasoning: Storing database credentials (username/password) in plain text YAML files is a security vulnerability. Secrets encode this data and inject it securely as environment variables at runtime.

Headless Service

Choice: ClusterIP: None (Headless Service) for MongoDB.

Reasoning: This allows the application to resolve the IP address of the specific MongoDB pod instance directly, rather than going through a load-balancing IP. This is often preferred for stateful applications where direct node communication is necessary.

Testing Scenarios and Autoscaling Results

1. Database Connectivity Test

To verify the application could communicate with the database, the following tests were performed:

Action: Sent a POST request to /data with sample JSON.

Result: Received 201 Created, verifying the app could write to MongoDB.

Action: Sent a GET request to /data.

Result: Received the saved JSON data, verifying read persistence.

2. Autoscaling (HPA) Stress Test

Objective: Verify that the Horizontal Pod Autoscaler scales the application from 2 pods to 3+ pods when CPU usage exceeds 70%.

Methodology:
A shell loop was used to generate high traffic and spike the CPU usage:

while true; do curl -X POST -H "Content-Type: application/json" -d '{"loadtest": "true"}' http://<MINIKUBE-IP>:<PORT>/data; done


Results:

Baseline: The application started with 2 replicas at <1% CPU.

Load Applied: CPU usage spiked to 90% (exceeding the 70% threshold).

Scaling Event: The HPA detected the spike and increased the replica count to 3.

Proof of Scaling:
(Please refer to the attached screenshot Autoscaling_Proof.png showing the REPLICAS count increasing from 2 to 3 under load)
