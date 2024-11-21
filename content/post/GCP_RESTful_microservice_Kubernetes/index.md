---
title: Kubernetes `SampleApp` Microservice Deployment on GCP with Terraform 🚀
date: 2024-05-03
description: This project demonstrates deploying a RESTful microservice called 'SampleApp' to a Google Kubernetes Engine (GKE) cluster in GCP. The service retrieves the current date/time from a Cloud SQL database (accessed via Cloud SQL Auth Proxy). I used Terraform to provision GCP resources. Docker is used for containerization, with the image stored in Artifact Registry. Kubernetes manifests manage deployment, and a load balancer routes incoming traffic.
categories: ["Cloud","DevOps","IaC","Kubernetes","Development"]
tag: ["terraform","GCP","K8s","Docker","Python"]
image: "architecture-diagram.png"
---

- [Prerequisites 🔧](#prerequisites-)
- [4. Steps: 🚀](#4-steps-)
  - [4.1. Step 1: Set up the GCP project](#41-step-1-set-up-the-gcp-project)
  - [4.2. Step 2: Python Flask Microservice 🐍](#42-step-2-python-flask-microservice-)
  - [4.3. Step 3: Dockerfile and the docker image 🐳](#43-step-3-dockerfile-and-the-docker-image-)
  - [4.4. Step 4: Kubernetes manifest files 📄](#44-step-4-kubernetes-manifest-files-)
  - [4.5. Step 5: Cloud SQL Auth Proxy 🔒](#45-step-5-cloud-sql-auth-proxy-)
  - [4.6. Step 6: Testing and troubleshooting 🧪](#46-step-6-testing-and-troubleshooting-)

## Prerequisites 🔧

**Cloud Resources:**

- [Google Cloud Platform](https://console.cloud.google.com/welcome?project=microservice-on-kubernetes) account and project
- Service accounts with required permissions
- [gcloud CLI](https://cloud.google.com/sdk/docs/install) and SDK installed
- Enabled APIs: Cloud SQL Admin API, Kubernetes Engine API, Artifact Registry API, IAM Service Account Credentials API
- Cloud SQL instance, database, and user
- Artifact Registry for storing the Docker image
- Google Kubernetes Engine cluster
- Cloud SQL Auth Proxy for connecting to the Cloud SQL instance from the GKE cluster

**Development Environment:**

- Code editor: [VS Code](https://code.visualstudio.com/download) (optional)
- [Terraform](https://developer.hashicorp.com/terraform/install) for provisioning infrastructure
- [Python](https://www.python.org/downloads/windows/) (for building the Flask microservice)
- [Docker](https://www.docker.com/products/docker-desktop/) for containerization

**Kubernetes Tools:**

- [kubectl](https://kubernetes.io/docs/tasks/tools/) for interacting with the Kubernetes cluster

## 4. Steps: 🚀

### 4.1. Step 1: Set up the GCP project

- Create a new project in GCP
- Create a service account with the required permissions (e.g. Storage Admin, Kubernetes Engine Admin, Artifact Registry Admin, Service Account User)
    - Add a key to the service account and download the JSON file 
- Enable the necessary APIs: Cloud SQL Admin API, Kubernetes Engine API, Artifact Registry API, IAM Service Account Credentials API
- Create the Terraform files: 
        ```
        provider.tf - GCP provider configuration
        variables.tf - Input variables for the Terraform configuration
        main.tf - Terraform configuration for creating the Cloud SQL instance, database, and user
        terraform.tfvars - Variable values for the Terraform configuration
        ```
**NOTE:** Make sure to include sensitive information in your gitignore file and do not expose them in the main code or in GitHub.

- Create a Google Kubernetes Engine cluster
    - **main.tf** - Terraform configuration for creating the GKE cluster
    
- Create an Artifact Registry repository in GCP and push the Docker image to the registry.
    - **main.tf** - Terraform configuration for creating the Artifact Registry repository

- Run `gcloud auth activate-service-account --key-file=[KEY_FILE]` to authenticate the service account
- Run `terraform init` to initialize the Terraform configuration
- Run `terraform plan` to view the resources that will be created
- Run `terraform apply` to create the resources

### 4.2. Step 2: Python Flask Microservice 🐍

- Build a simple Python Flask microservice (using Terraform) that retrieves the current date/time from a Cloud SQL database
    
    - **sample-app.py:** Python Flask code for the microservice
    - **requirements.txt:** Required Python packages (e.g. Flask, Flask-SQLAlchemy, MySQL-connecttor-python)

### 4.3. Step 3: Dockerfile and the docker image 🐳

    - **Dockerfile:** Configuration for building the Docker image
- Run `gcloud auth configure-docker` to authenticate Docker to the Artifact Registry
- Run `docker build -t [HOSTNAME]/[PROJECT-ID]/[REPOSITORY]/[IMAGE]:[TAG] .` to build and tag the Docker image
- Run `docker push [HOSTNAME]/[PROJECT-ID]/[REPOSITORY]/[IMAGE]:[TAG]` to push the Docker image to the Artifact Registry

### 4.4. Step 4: Kubernetes manifest files 📄

- Create the Kubernetes manifest files for the deployment, service, and Kubernetes service account (KSA)

    - **kubernetes-deployment-manifest.yaml:** Deployment configuration for the microservice
    - **kubernetes-loadbalancer.yaml:** Service configuration for exposing the microservice
    - **kubernetes-service-account.yaml:** Kubernetes service account configuration for the microservice

- Run `gcloud container clusters get-credentials [CLUSTER_NAME] --zone [ZONE] --project [PROJECT_ID]` to authenticate kubectl to the GKE cluster
- Run `kubectl apply -f kubernetes-deployment-manifest.yaml` to deploy the microservice
- Run `kubectl apply -f kubernetes-loadbalancer.yaml` to expose the microservice (I use LoadBalancer)
- Run `kubectl apply -f kubernetes-service-account.yaml` to create the Kubernetes service account

- Kubernetes secrets(to store the Cloud SQL credentials):
    - **main.tf:** Terraform configuration for creating the Kubernetes secret

### 4.5. Step 5: Cloud SQL Auth Proxy 🔒

- The Cloud SQL Auth Proxy is a Cloud SQL connector that provides secure access to your instances without a need for Authorized networks or for configuring SSL. When you connect using the Cloud SQL Auth Proxy, the Cloud SQL Auth Proxy is added to your pod using the sidecar container pattern. The Cloud SQL Auth Proxy container is in the same pod as your application, which enables the application to connect to the Cloud SQL Auth Proxy using localhost, increasing security and performance.

I used the method **Workload Identity** to bind a KSA to a GSA, causing any deployments with that KSA to authenticate as the GSA in their interactions with Google Cloud. GKE Autopilot cluster has Workload Identity enabled by default.

- Create a Google service account (GSA) with the required permissions (e.g. Cloud SQL Client)
- Bind the KSA to the GSA by this command:
    ```
    gcloud iam service-accounts add-iam-policy-binding \
    --role roles/iam.workloadIdentityUser \
    --member "serviceAccount:[PROJECT_ID].svc.id.goog[NAMESPACE]/[KSA_NAME]" \
    [GSA_EMAIL]
    ```

### 4.6. Step 6: Testing and troubleshooting 🧪

- Test the Kubernetes deployment and the pods by running `kubectl get deployments` and `kubectl get pods`.
- Test the microservice by sending a GET request to the load balancer IP address
    - `curl <EXTERNAL IP ADDRESS>`
    - <http://EXTERNAL IP ADDRESS>
    - GET http://EXTERNAL IP ADDRESS 
