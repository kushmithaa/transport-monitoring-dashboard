# Public Transport Monitoring Dashboard

## Project Overview

The Public Transport Monitoring Dashboard is a web-based application designed to display simulated public transport information in a simple and user-friendly interface. It helps users monitor bus routes, estimated arrival times, crowd levels, service status, and service disruptions.

This project demonstrates the use of Git, GitHub, Docker, Jenkins, and Kubernetes as part of an Agile Development Process and DevOps workflow.

## Features

- Display simulated bus route information.
- Show route numbers, source, destination, and estimated arrival times.
- Display passenger crowd levels and service status.
- Filter routes using route and service-status filters.
- Display service disruption alerts and travel advisories.
- Provide a responsive dashboard interface.

## Technologies Used

- **Frontend:** HTML, CSS, JavaScript
- **Version Control:** Git and GitHub
- **Containerization:** Docker
- **CI/CD:** Jenkins
- **Container Orchestration:** Kubernetes using Docker Desktop
- **Container Registry:** Docker Hub
- **Web Server:** Nginx

## Project Structure

```text
transport-monitoring-dashboard/
├── index.html
├── Dockerfile
├── Jenkinsfile
├── deployment.yaml
└── README.md
```

## Prerequisites

Install and configure the following tools:

- Git
- Docker Desktop
- Jenkins
- kubectl
- A GitHub account
- A Docker Hub account

The Jenkins agent must have access to Docker and kubectl to execute the pipeline and deploy the application.

## Run the Application Locally

### 1. Clone the repository

```bash
git clone https://github.com/kushmithaa/transport-monitoring-dashboard.git
cd transport-monitoring-dashboard
```

### 2. Build the Docker image

```bash
docker build -t transport-monitoring-dashboard:latest .
```

### 3. Run the container

```bash
docker run -d --name transport-dashboard -p 8080:80 transport-monitoring-dashboard:latest
```

### 4. Open the dashboard

Visit the following address in your browser:

http://localhost:8080

To stop and remove the container:

```bash
docker stop transport-dashboard
docker rm transport-dashboard
```

## Push the Image to Docker Hub

Log in to Docker Hub:

```bash
docker login
```

Tag the image:

```bash
docker tag transport-monitoring-dashboard:latest kushmithaa/transport-monitoring-dashboard:latest
```

Push the image:

```bash
docker push kushmithaa/transport-monitoring-dashboard:latest
```

## Jenkins CI/CD Pipeline

The `Jenkinsfile` defines an automated pipeline for building, validating, and deploying the application.

The pipeline demonstrates:

- **Parallel stages:** Run independent validation tasks concurrently.
- **Parameterized builds:** Select build options through Jenkins parameters.
- **Conditional execution:** Execute stages based on selected parameters.
- **Docker integration:** Build and publish container images.
- **Kubernetes deployment:** Deploy the application and verify rollout status.
- **Automatic rollback:** Attempt to restore the previous stable deployment if a rollout fails.

### Jenkins Configuration

Configure a Jenkins Pipeline job using the repository URL:

```text
https://github.com/kushmithaa/transport-monitoring-dashboard.git
```

Ensure that the Jenkins agent has Docker CLI access and that Docker Desktop is running. Configure the required Docker Hub credentials and Kubernetes access in Jenkins before running deployment stages.

Use the parameter options provided by the `Jenkinsfile`.

## Kubernetes Deployment

Ensure Docker Desktop Kubernetes is enabled and `kubectl` can access the cluster.

Apply the deployment manifest:

```bash
kubectl apply -f deployment.yaml
```

Check the deployment:

```bash
kubectl get deployments
kubectl get pods
kubectl get services
```

Check the rollout status:

```bash
kubectl rollout status deployment/transport-dashboard
```

The application is configured to use a NodePort service on port `30010`. When the Kubernetes deployment and service are running, open:

http://localhost:30010

## Rollback Testing

To demonstrate deployment failure and recovery:

1. Confirm that a stable version of the application is deployed.
2. Introduce a controlled readiness-probe failure in `deployment.yaml` by temporarily changing the probe path from `/` to `/missing-health-check`.
3. Commit and push the change to GitHub.
4. Run the Jenkins pipeline using the appropriate release parameter.
5. Observe the failed Kubernetes rollout and check the Jenkins console output for rollback activity.
6. Verify the deployment and pods after rollback.
7. Restore the readiness-probe path to `/`, commit and push the fix, and run a stable deployment.

**Note:** A failed rollout can only trigger the intended rollback if the Jenkins pipeline's failure-handling logic executes successfully and the previous stable ReplicaSet is available.

## Git Commit History

Use the following commit messages for the initial project and enhancement:

**Initial commit**

```text
Initial commit: create public transport monitoring dashboard
```

**Enhancement commit**

```text
Add route filtering and service disruption alerts
```

View the project history using:

```bash
git log --oneline
```

## Expected Outcome

The completed project demonstrates a public transport monitoring dashboard with route filtering and disruption alerts, containerized using Docker, automated through Jenkins, published to Docker Hub, and deployed to Kubernetes with rollout verification and rollback testing.

## Author

GitHub: https://github.com/kushmithaa

Repository: https://github.com/kushmithaa/transport-monitoring-dashboard
