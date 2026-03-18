# How to Run Jenkins in a Podman Container

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Containers, DevOps, Jenkins, CI/CD, Automation

Description: Learn how to run Jenkins in a Podman container with persistent workspace data, plugin management, and pipeline configuration.

---

> Jenkins in Podman provides a fully-featured CI/CD server in a rootless container with persistent jobs, plugins, and build history.

Jenkins is the most widely adopted open-source automation server, powering CI/CD pipelines for teams of all sizes. Running it in a Podman container gives you a portable, isolated Jenkins instance that is easy to back up, upgrade, and replicate. This guide walks through setup, persistence, initial configuration, and plugin management.

---

## Pulling the Jenkins Image

Download the official Jenkins LTS image.

```bash
# Pull the Jenkins LTS image
podman pull docker.io/jenkins/jenkins:lts

# Verify the image
podman images | grep jenkins
```

## Running a Basic Jenkins Container

Start Jenkins with the default setup wizard.

```bash
# Run Jenkins in detached mode
podman run -d \
  --name my-jenkins \
  -p 8080:8080 \
  -p 50000:50000 \
  jenkins/jenkins:lts

# Check the container is running
podman ps

# Retrieve the initial admin password for the setup wizard
podman exec my-jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

## Persistent Jenkins Data

Use a named volume to preserve jobs, plugins, and configuration.

```bash
# Create a volume for Jenkins home directory
podman volume create jenkins-data

# Run Jenkins with persistent storage
podman run -d \
  --name jenkins-persistent \
  -p 8081:8080 \
  -p 50001:50000 \
  -v jenkins-data:/var/jenkins_home:Z \
  jenkins/jenkins:lts

# Verify the volume
podman volume inspect jenkins-data
```

## Skipping the Setup Wizard

Pre-configure Jenkins to skip the manual setup wizard.

```bash
# Run Jenkins with the setup wizard disabled
podman run -d \
  --name jenkins-auto \
  -p 8082:8080 \
  -e JAVA_OPTS="-Djenkins.install.runSetupWizard=false" \
  -v jenkins-data:/var/jenkins_home:Z \
  jenkins/jenkins:lts

# Wait for Jenkins to start
sleep 30

# Verify Jenkins is running and accessible
curl -s http://localhost:8082/api/json | python3 -m json.tool | head -10
```

## Installing Plugins at Startup

Pre-install Jenkins plugins using the install-plugins script.

```bash
# Create a plugins list file
mkdir -p ~/jenkins-config

cat > ~/jenkins-config/plugins.txt <<'EOF'
git
pipeline-stage-view
docker-workflow
blueocean
credentials
workflow-aggregator
github
slack
EOF

# Build a custom Jenkins image with pre-installed plugins
cat > ~/jenkins-config/Containerfile <<'EOF'
FROM jenkins/jenkins:lts

# Skip the setup wizard
ENV JAVA_OPTS="-Djenkins.install.runSetupWizard=false"

# Install plugins
COPY plugins.txt /usr/share/jenkins/ref/plugins.txt
RUN jenkins-plugin-cli --plugin-file /usr/share/jenkins/ref/plugins.txt
EOF

# Build the custom image
podman build -t jenkins-custom -f ~/jenkins-config/Containerfile ~/jenkins-config

# Run the custom Jenkins image
podman run -d \
  --name jenkins-plugins \
  -p 8083:8080 \
  -v jenkins-data:/var/jenkins_home:Z \
  jenkins-custom
```

## Configuring Jenkins with JCasC

Use Jenkins Configuration as Code for automated setup.

```bash
# Create a JCasC configuration file
cat > ~/jenkins-config/jenkins.yaml <<'EOF'
jenkins:
  systemMessage: "Jenkins configured via JCasC on Podman"
  numExecutors: 4
  securityRealm:
    local:
      allowsSignup: false
      users:
        - id: "admin"
          password: "admin-secret"
  authorizationStrategy:
    loggedInUsersCanDoAnything:
      allowAnonymousRead: false

unclassified:
  location:
    url: http://localhost:8084
EOF

# Run Jenkins with JCasC configuration
podman run -d \
  --name jenkins-jcasc \
  -p 8084:8080 \
  -e JAVA_OPTS="-Djenkins.install.runSetupWizard=false" \
  -e CASC_JENKINS_CONFIG=/var/jenkins_home/casc_configs \
  -v ~/jenkins-config/jenkins.yaml:/var/jenkins_home/casc_configs/jenkins.yaml:Z \
  -v jenkins-data:/var/jenkins_home:Z \
  jenkins/jenkins:lts
```

## Running Jenkins with Resource Limits

Constrain Jenkins memory and CPU usage.

```bash
# Run Jenkins with resource limits
podman run -d \
  --name jenkins-limited \
  -p 8085:8080 \
  --memory 2g \
  --cpus 2.0 \
  -e JAVA_OPTS="-Xms512m -Xmx1g -Djenkins.install.runSetupWizard=false" \
  -v jenkins-data:/var/jenkins_home:Z \
  jenkins/jenkins:lts
```

## Managing the Container

Common Jenkins management operations.

```bash
# View Jenkins logs
podman logs my-jenkins

# Safely restart Jenkins via the API
curl -X POST http://localhost:8080/safeRestart

# Check Jenkins system info
curl -s http://localhost:8080/api/json | python3 -m json.tool | head -15

# Stop and start
podman stop my-jenkins
podman start my-jenkins

# Remove containers and volumes
podman rm -f my-jenkins jenkins-persistent jenkins-auto jenkins-plugins jenkins-jcasc
podman volume rm jenkins-data
```

## Summary

Running Jenkins in a Podman container provides a portable CI/CD server with straightforward configuration and management. Named volumes preserve your jobs, build history, and plugins across container restarts. Pre-installing plugins and using Jenkins Configuration as Code eliminates manual setup steps, making your Jenkins instance fully reproducible. Resource limits keep Jenkins from consuming excessive host resources, and Podman's rootless execution adds a security boundary around your automation server.
