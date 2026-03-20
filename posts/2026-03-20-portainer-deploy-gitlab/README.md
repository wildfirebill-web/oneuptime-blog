# How to Deploy GitLab via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, GitLab, CI/CD, DevOps, Self-Hosted, Git

Description: Deploy GitLab CE via Portainer for a complete self-hosted DevSecOps platform including Git hosting, CI/CD pipelines, container registry, and issue tracking.

## Introduction

GitLab CE (Community Edition) provides everything a development team needs: Git hosting, merge requests, CI/CD pipelines, container registry, and issue tracking. Deploying via Portainer gives you a manageable setup with persistent storage.

## Prerequisites

- Docker host with at least 8GB RAM (GitLab is resource-intensive)
- 10GB+ available disk space
- A domain name (recommended)

## Deploy as a Stack

```yaml
version: "3.8"

services:
  gitlab:
    image: gitlab/gitlab-ce:latest
    container_name: gitlab
    hostname: gitlab.example.com  # Change to your domain
    environment:
      GITLAB_OMNIBUS_CONFIG: |
        external_url 'http://gitlab.example.com'
        gitlab_rails['gitlab_shell_ssh_port'] = 2224
        gitlab_rails['time_zone'] = 'UTC'
        
        # Reduce memory usage
        puma['worker_processes'] = 2
        puma['max_threads'] = 4
        sidekiq['max_concurrency'] = 5
        
        # Email settings
        gitlab_rails['smtp_enable'] = true
        gitlab_rails['smtp_address'] = 'smtp.example.com'
        gitlab_rails['smtp_port'] = 587
        gitlab_rails['smtp_user_name'] = 'gitlab@example.com'
        gitlab_rails['smtp_password'] = 'smtp_password'
        gitlab_rails['smtp_domain'] = 'example.com'
        gitlab_rails['smtp_authentication'] = 'login'
        gitlab_rails['smtp_enable_starttls_auto'] = true
        
        # Container registry
        registry_external_url 'http://registry.example.com'
    volumes:
      - gitlab_config:/etc/gitlab
      - gitlab_logs:/var/log/gitlab
      - gitlab_data:/var/opt/gitlab
    ports:
      - "80:80"
      - "443:443"
      - "2224:22"     # SSH port (using non-standard to avoid host conflict)
    shm_size: '256m'   # Required for GitLab
    restart: unless-stopped

volumes:
  gitlab_config:
  gitlab_logs:
  gitlab_data:
```

## Initial Access

After deployment (allow 2-5 minutes for initialization):

```bash
# Get initial root password
docker exec gitlab grep 'Password:' /etc/gitlab/initial_root_password
```

Navigate to `http://gitlab.example.com` and log in with `root` and the initial password.

## GitLab CI/CD Pipeline Example

Create `.gitlab-ci.yml` in your repository:

```yaml
# GitLab CI pipeline
stages:
  - build
  - test
  - push
  - deploy

variables:
  DOCKER_REGISTRY: registry.example.com
  IMAGE_NAME: myapp

build:
  stage: build
  image: docker:24
  services:
    - docker:24-dind
  script:
    - docker build -t ${DOCKER_REGISTRY}/${IMAGE_NAME}:${CI_COMMIT_SHA} .
    - docker tag ${DOCKER_REGISTRY}/${IMAGE_NAME}:${CI_COMMIT_SHA} ${DOCKER_REGISTRY}/${IMAGE_NAME}:latest
  only:
    - main

test:
  stage: test
  image: ${DOCKER_REGISTRY}/${IMAGE_NAME}:${CI_COMMIT_SHA}
  script:
    - npm test
  only:
    - main

push:
  stage: push
  image: docker:24
  services:
    - docker:24-dind
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $DOCKER_REGISTRY
    - docker push ${DOCKER_REGISTRY}/${IMAGE_NAME}:${CI_COMMIT_SHA}
    - docker push ${DOCKER_REGISTRY}/${IMAGE_NAME}:latest
  only:
    - main

deploy:
  stage: deploy
  image: curlimages/curl:latest
  script:
    # Trigger Portainer webhook to redeploy
    - curl -X POST ${PORTAINER_WEBHOOK_URL}
  only:
    - main
```

## GitLab Runner for Portainer-Deployed Apps

Deploy a GitLab Runner to handle CI jobs:

```yaml
version: "3.8"

services:
  gitlab-runner:
    image: gitlab/gitlab-runner:latest
    container_name: gitlab-runner
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - gitlab_runner_config:/etc/gitlab-runner
    restart: unless-stopped

volumes:
  gitlab_runner_config:
```

Register the runner:

```bash
docker exec -it gitlab-runner gitlab-runner register \
  --url http://gitlab.example.com \
  --registration-token YOUR_REGISTRATION_TOKEN \
  --executor docker \
  --docker-image docker:24 \
  --docker-volumes /var/run/docker.sock:/var/run/docker.sock
```

## Conclusion

GitLab CE deployed via Portainer provides a complete self-hosted DevSecOps platform. The persistent volumes store your repositories, pipelines, and configurations safely. The integrated container registry and CI/CD pipelines make it possible to build complete continuous deployment workflows, with Portainer webhooks providing the deployment trigger.
