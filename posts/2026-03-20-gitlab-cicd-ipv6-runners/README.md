# How to Configure GitLab CI/CD with IPv6 Runners

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, GitLab, CI/CD, Runner, Docker, DevOps

Description: Configure GitLab CI/CD runners to use IPv6 networking, enable IPv6 in Docker executor builds, and test IPv6 connectivity in GitLab CI pipelines.

## Introduction

GitLab CI/CD runners support IPv6 for both the runner registration communication and the build environment. Enabling IPv6 in GitLab CI requires configuring the runner's `config.toml`, enabling IPv6 in Docker networks, and ensuring the GitLab instance itself is reachable over IPv6.

## Step 1: Configure GitLab Runner for IPv6

```toml
# /etc/gitlab-runner/config.toml

concurrent = 4
check_interval = 0

[session_server]
  session_timeout = 1800

[[runners]]
  name = "ipv6-docker-runner"
  url = "https://gitlab.example.com"
  token = "<your-registration-token>"
  executor = "docker"

  [runners.docker]
    tls_verify = false
    image = "ubuntu:22.04"
    privileged = false
    disable_entrypoint_overwrite = false
    oom_kill_disable = false
    disable_cache = false
    volumes = ["/cache"]

    # Enable IPv6 in Docker containers for CI builds
    network_mode = "ipv6-ci-net"

    # Or configure IPv6 directly
    [[runners.docker.sysctls]]
      "net.ipv6.conf.all.disable_ipv6" = "0"
```

## Step 2: Create IPv6-Enabled Docker Network for Builds

```bash
# Create a Docker network with IPv6 support for GitLab CI builds
docker network create \
    --driver bridge \
    --ipv6 \
    --subnet 2001:db8:ci::/64 \
    --gateway 2001:db8:ci::1 \
    ipv6-ci-net

# Verify network was created with IPv6
docker network inspect ipv6-ci-net | python3 -m json.tool | grep -A5 IPAM
```

Or add to `/etc/docker/daemon.json`:

```json
{
  "ipv6": true,
  "fixed-cidr-v6": "2001:db8:1::/64",
  "ip6tables": true,
  "experimental": true
}
```

## Step 3: Register the Runner with IPv6

```bash
# Register the runner, ensuring it can reach GitLab over IPv6
gitlab-runner register \
    --url https://[2001:db8::gitlab]/  \
    --registration-token <token> \
    --executor docker \
    --docker-image ubuntu:22.04 \
    --description "IPv6 Docker Runner" \
    --tag-list ipv6,docker \
    --run-untagged=false \
    --locked=false
```

## Step 4: GitLab CI Pipeline with IPv6 Tests

```yaml
# .gitlab-ci.yml

variables:
  DOCKER_BUILDKIT: "1"

stages:
  - test
  - build
  - deploy

ipv6-connectivity-test:
  stage: test
  image: ubuntu:22.04
  tags:
    - ipv6
  script:
    # Install networking tools
    - apt-get update -q && apt-get install -y iputils-ping curl dnsutils
    # Test IPv6 connectivity from the CI environment
    - ip -6 addr show
    - ping6 -c 3 2606:4700:4700::1111
    - curl -6 https://ipv6.icanhazip.com
    - dig AAAA example.com +short

build-with-ipv6:
  stage: build
  image: docker:24
  services:
    - docker:24-dind
  tags:
    - ipv6
  variables:
    DOCKER_HOST: tcp://docker:2376
    DOCKER_TLS_CERTDIR: "/certs"
  script:
    # Build Docker image - base images pulled over IPv6 if available
    - docker build -t myapp:$CI_COMMIT_SHA .
    # Test IPv6 connectivity in the built container
    - docker run --rm myapp:$CI_COMMIT_SHA curl -6 https://example.com || true

deploy-to-ipv6-cluster:
  stage: deploy
  tags:
    - ipv6
  script:
    # Deploy to Kubernetes cluster with IPv6 addresses
    - kubectl apply -f k8s/
    # Wait for rollout and verify IPv6 service
    - kubectl rollout status deployment/myapp
    - kubectl get svc myapp -o jsonpath='{.status.loadBalancer.ingress}'
  only:
    - main
```

## Step 5: GitLab Runner Docker Network IPv6 Verification

```bash
# Verify that CI containers get IPv6 addresses
docker run --rm --network ipv6-ci-net ubuntu:22.04 ip -6 addr show

# Test connectivity from a CI-like container
docker run --rm --network ipv6-ci-net ubuntu:22.04 \
    sh -c "apt-get update -q && apt-get install -y iputils-ping && ping6 -c 3 2606:4700:4700::1111"
```

## Troubleshooting

```bash
# If containers don't get IPv6 addresses:
# 1. Check Docker daemon has IPv6 enabled
docker info | grep IPv6

# 2. Check ip6tables rules are not blocking
ip6tables -L DOCKER -n

# 3. Check that the network has IPv6 configured
docker network inspect ipv6-ci-net

# 4. Ensure kernel has IPv6 enabled
sysctl net.ipv6.conf.all.disable_ipv6
# Must be 0
```

## Conclusion

GitLab CI/CD runners support IPv6 by configuring `config.toml` to use IPv6-enabled Docker networks and ensuring the Docker daemon has IPv6 enabled. The combination of a custom Docker network with a /64 IPv6 prefix and ip6tables masquerading allows CI build containers to have full IPv6 connectivity. This enables testing IPv6-specific features, pulling dependencies over IPv6, and deploying to IPv6-only or dual-stack infrastructure directly from CI pipelines.
