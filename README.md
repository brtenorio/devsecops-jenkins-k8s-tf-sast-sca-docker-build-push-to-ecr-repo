# Flask App — Build, Run, CI/CD

This repo contains a Flask application (in `flask-app/`) with a Dockerfile and a Jenkins pipeline to build an image and push to AWS ECR.

## Contents
- `flask-app/` — Flask application source, `requirements.txt` and Dockerfile
- `Jenkinsfile` — Declarative pipeline: build from `flask-app/`, tag for ECR, push (uses AWS CLI login)
- `Dockerfile` at repo root — (legacy Java app); ignore for Flask deployment

## Prerequisites (local / CI agent)
- Docker Desktop (Mac)
- Python 3.x (for local dev)
- If using Jenkins to push to ECR: aws-cli and Docker available on the agent
- AWS credentials/permissions to create/push to ECR repo

## Local: build and run (quick)
1. From repo root build the image (context is `flask-app/`):
```bash
docker build -t flask-app:local ./flask-app
```

2. Run mapping host port 5010 to container port 5000 (container listens on 5000):
```bash
docker run --rm -p 5010:5000 --name flask-local flask-app:local
```