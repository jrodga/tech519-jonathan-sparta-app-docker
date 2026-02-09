# CI with GitHub Actions – Sparta App (Node.js + MongoDB)

This document describes the **Continuous Integration (CI)** setup for the **Sparta Node.js** application, which uses **Express** as the web framework and MongoDB as the database.

This CI documentation is kept separate from the main README to avoid making the core project documentation too long.
##

### Purpose of This CI Pipeline

The GitHub Actions pipeline is used to:

- Automatically run when code is pushed to the `main` branch

- Build the Sparta application Docker image

- Validate that the application builds correctly

- Prepare the project for future deployment (CD)

This ensures that changes pushed to the repository do not break the application build.
##
### CI Scope (Important)

This CI pipeline focuses on the **application container** only.

- MongoDB is **not started** in CI

- Database seeding is **not required** for CI

- The goal is to verify:

    - Dockerfile validity

    - Application build

    - Container startup

This matches common DevOps practice where CI validates builds, not full system orchestration.

##
### Workflow Trigger
The pipeline is triggered on every push to the `main` branch:

```yml
on:
  push:
    branches:
      - main
```
This follows standard CI practice where the main branch always represents a stable state.
##
### Docker Hub Authentication
The pipeline logs into Docker Hub using GitHub Secrets.

Secrets used:

`DOCKERHUB_USERNAME`

`DOCKERHUB_TOKEN`

Authentication is handled using the official Docker GitHub Action:
```yml
- name: Login to DockerHub
  uses: docker/login-action@v3
  with:
    username: ${{ secrets.DOCKERHUB_USERNAME }}
    password: ${{ secrets.DOCKERHUB_TOKEN }}
```
This avoids interactive login, which is not supported in CI environments.
##
### Build and Test Stages
**Build Docker Image**

The application image is built using the development Dockerfile:
```yml
- name: Build Docker image
  run: docker build -t jrodga1604/sparta-app .
```
This step verifies that:

- Dependencies install correctly

- The Dockerfile is valid

- The application can be containerised successfully
##
### Optional: Container Startup Check

To ensure the container starts correctly, the image can be run briefly:
```yml
- name: Run container (sanity check)
  run: docker run -d -p 3000:3000 jrodga1604/sparta-app
```
This confirms:

- The Node.js app starts

- No runtime errors occur on launch

Database connectivity is not tested here, as MongoDB runs in Docker Compose during development and deployment.
##
### Important Note: Updating a Failing Workflow

When a workflow fails and the YAML file is updated, GitHub Actions does not automatically re-run the job using the new configuration.

This is because:

- Each workflow run uses the workflow file from the commit that triggered it

- Re-running a failed job may still use the old configuration
##

### How to Apply Workflow Changes Correctly

To ensure GitHub Actions uses the **latest workflow version**, a new commit must be pushed.

**Recommended approach**

Make any small change (for example updating documentation), then:

```bash
git add .
git commit -m "Update CI workflow"
git push
```
This triggers a new workflow run using the updated **YAML** file.