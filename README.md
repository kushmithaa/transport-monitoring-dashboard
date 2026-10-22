# Public Transport Monitoring Dashboard

Assessment project using GitHub, Jenkins, Docker, Docker Hub and Docker Desktop Kubernetes.

## Git commits required
Create a GitHub repository named `transport-monitoring-dashboard`, then run in this folder:

```bat
git init
git branch -M main
git config user.name "kushmithaa"
git add index.html Dockerfile Jenkinsfile deployment.yaml README.md
git commit -m "Initial commit: create public transport monitoring dashboard"
git remote add origin https://github.com/kushmithaa/transport-monitoring-dashboard.git
git push -u origin main
```

After modifying routes/advisories:
```bat
git add index.html
git commit -m "Add route filtering and service disruption alerts"
git push
```
If you created a different repository name, use its actual GitHub URL.

## Local Docker
```bat
docker build -t kushmithaa/transport-monitoring-dashboard:stable .
docker run -d --name transport-dashboard-local -p 8085:80 kushmithaa/transport-monitoring-dashboard:stable
```
Open http://localhost:8085. Cleanup: `docker stop transport-dashboard-local` then `docker rm transport-dashboard-local`.

## Docker Hub
```bat
docker login
docker push kushmithaa/transport-monitoring-dashboard:stable
```

## Kubernetes
Enable Kubernetes in Docker Desktop, then:
```bat
kubectl apply -f deployment.yaml
kubectl rollout status deployment/transport-dashboard
kubectl get pods -l app=transport-dashboard
kubectl get service transport-dashboard-service
```
Open http://localhost:30010.

## Jenkins setup
Use a Windows Jenkins agent with Docker CLI/daemon and kubectl installed.
Create credentials:
- `dockerhub`: Username/password credential; username `kushmithaa`, password a Docker Hub access token.
- `kubeconfig`: Secret file containing kubeconfig for the Docker Desktop Kubernetes cluster.

Run the job as a Pipeline from SCM pointing at the GitHub repository and `Jenkinsfile`.
- `PIPELINE_TYPE=parallel`: parallel HTML and manifest validation.
- `PIPELINE_TYPE=conditional`: conditional advisory validation.
- `RELEASE_MODE=stable`: build, smoke-test, publish and deploy a stable image.
- `RELEASE_MODE=faulty`: publish an intentionally faulty tag and attempt rollout. To reliably force a readiness failure, first deliberately change the readiness probe path in `deployment.yaml` from `/` to `/missing-health-check`, commit/push that change, then run the faulty release. Confirm rollout fails and `kubectl rollout undo` restores the previous ReplicaSet. Revert the probe path to `/`, commit and push the fix, and run a stable build.

Do a successful stable deployment first so there is a known-good ReplicaSet to restore. Never commit Docker Hub tokens or kubeconfig files.
