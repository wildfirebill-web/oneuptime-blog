# How to Configure CircleCI with IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, CircleCI, CI/CD, Docker, DevOps, Testing

Description: Configure CircleCI pipelines for IPv6 testing using self-hosted runners, Docker networks with IPv6, and pipeline steps that verify IPv6 connectivity.

## Introduction

CircleCI's cloud executors have limited IPv6 support, but self-hosted runners (CircleCI Runners) can provide full IPv6 connectivity. This guide covers setting up a CircleCI Runner with IPv6, configuring Docker IPv6 networks for test containers, and writing CircleCI config.yml for IPv6 testing.

## Step 1: Set Up a Self-Hosted CircleCI Runner with IPv6

```bash
# Install CircleCI Runner on a host with IPv6 connectivity

# Follow CircleCI's runner installation guide for your OS

# Debian/Ubuntu installation
curl -fsSL https://packagecloud.io/install/repositories/circleci/runner/script.deb.sh | sudo bash
sudo apt-get install circleci-launch-agent

# Configure the runner
sudo mkdir -p /etc/opt/circleci
sudo tee /etc/opt/circleci/launch-agent-config.yaml > /dev/null << EOF
api:
  auth_token: <your-runner-token>

runner:
  name: ipv6-runner-$(hostname)
  working_directory: /var/opt/circleci/workdir
  cleanup_working_directory: true
EOF

# Start the runner
sudo systemctl enable --now circleci.service

# Verify the runner has IPv6
ip -6 addr show scope global
ping6 -c 3 2606:4700:4700::1111
```

## Step 2: Enable Docker IPv6 on the Runner Host

```bash
# Configure Docker daemon for IPv6 on the runner host
sudo tee /etc/docker/daemon.json > /dev/null << 'EOF'
{
  "ipv6": true,
  "fixed-cidr-v6": "2001:db8:ci::/64",
  "ip6tables": true,
  "experimental": true
}
EOF

sudo systemctl restart docker

# Verify Docker has IPv6
docker info | grep -i ipv6
```

## Step 3: CircleCI config.yml with IPv6

```yaml
# .circleci/config.yml

version: 2.1

orbs:
  python: circleci/python@2.1.1

executors:
  ipv6-executor:
    machine:
      image: ubuntu-2204:current
    resource_class: <your-namespace>/<your-runner-name>

jobs:
  test-ipv6-connectivity:
    executor: ipv6-executor
    steps:
      - checkout

      - run:
          name: Verify IPv6 availability
          command: |
            ip -6 addr show scope global
            ping6 -c 3 2606:4700:4700::1111
            curl -6 https://api6.ipify.org

      - run:
          name: Create IPv6 Docker network
          command: |
            docker network create \
              --driver bridge \
              --ipv6 \
              --subnet 2001:db8:test::/64 \
              --gateway 2001:db8:test::1 \
              ipv6-test-net

      - run:
          name: Start test services with IPv6
          command: |
            docker run -d \
              --name test-server \
              --network ipv6-test-net \
              nginx:latest

            # Wait for container to start
            sleep 2

            # Get the IPv6 address of the container
            IPV6_ADDR=$(docker inspect test-server \
              -f '{{range .NetworkSettings.Networks}}{{.GlobalIPv6Address}}{{end}}')
            echo "Test server IPv6: $IPV6_ADDR"
            echo "export TEST_SERVER_IPV6=$IPV6_ADDR" >> $BASH_ENV

      - run:
          name: Run IPv6 integration tests
          command: |
            source $BASH_ENV
            # Test from another container on the IPv6 network
            docker run --rm --network ipv6-test-net \
              curlimages/curl:latest \
              curl -6 "http://[$TEST_SERVER_IPV6]/" -v

      - run:
          name: Cleanup
          command: |
            docker rm -f test-server || true
            docker network rm ipv6-test-net || true
          when: always

  build-and-push-ipv6:
    executor: ipv6-executor
    steps:
      - checkout
      - setup_remote_docker

      - run:
          name: Build application
          command: docker build -t myapp:$CIRCLE_SHA1 .

      - run:
          name: Test application IPv6 support
          command: |
            # Run the app and test IPv6 binding
            docker run -d --name myapp \
              -e LISTEN_ON_IPV6=true \
              myapp:$CIRCLE_SHA1

            # Verify app started and listening on IPv6
            docker logs myapp
            docker rm -f myapp

workflows:
  ipv6-pipeline:
    jobs:
      - test-ipv6-connectivity:
          filters:
            branches:
              only:
                - main
                - /feature\/.*/
      - build-and-push-ipv6:
          requires:
            - test-ipv6-connectivity
```

## Step 4: Testing Application IPv6 Binding in CircleCI

```yaml
# Additional job to test application-level IPv6 support
test-app-ipv6:
  executor: ipv6-executor
  steps:
    - checkout

    - run:
        name: Test Python application listens on IPv6
        command: |
          # Start the Python app configured for IPv6
          LISTEN_ADDR="::" python3 app.py &
          APP_PID=$!
          sleep 2

          # Verify it's listening on IPv6
          ss -6 -t -l -n | grep ":8080"

          # Test IPv6 connection
          curl -6 "http://[::1]:8080/health"

          kill $APP_PID
```

## Troubleshooting CircleCI IPv6

```bash
# If tests fail, check runner IPv6 status
sudo journalctl -u circleci -n 50

# Verify Docker IPv6 is working on the runner
docker run --rm ubuntu:22.04 ip -6 addr show

# Check if ip6tables masquerading is enabled for containers
sudo ip6tables -t nat -L POSTROUTING -n

# If missing, add masquerading for the Docker IPv6 subnet
sudo ip6tables -t nat -A POSTROUTING -s 2001:db8:ci::/64 ! -o docker0 -j MASQUERADE
```

## Conclusion

CircleCI IPv6 testing is most effective with a self-hosted runner configured with IPv6 access and Docker IPv6 enabled. The `config.yml` structure allows creating IPv6 Docker networks per job, running containers on those networks, and testing application IPv6 connectivity within the pipeline. For production use, ensure the runner host has a stable global IPv6 address and that Docker's ip6tables masquerading is properly configured for outbound IPv6 connectivity from containers.
