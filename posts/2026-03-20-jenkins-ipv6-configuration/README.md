# How to Configure Jenkins to Run on IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Jenkins, CI/CD, Java, Networking, DevOps

Description: Configure Jenkins to listen on IPv6 addresses, enable IPv6 agent communication, and test IPv6 connectivity within Jenkins pipeline jobs.

## Introduction

Jenkins is written in Java, which provides native IPv6 support through the JVM. Configuring Jenkins for IPv6 requires setting the appropriate JVM network flags, configuring the listener address, and ensuring agents can communicate over IPv6.

## Step 1: Configure Jenkins to Listen on IPv6

Jenkins by default listens on all interfaces. To explicitly bind to an IPv6 address:

```bash
# For Jenkins installed as a systemd service

# Edit the Jenkins service configuration
sudo systemctl edit jenkins

# Add the following override:
[Service]
Environment="JAVA_OPTS=-Djava.net.preferIPv6Addresses=true -Djava.net.preferIPv4Stack=false"
Environment="JENKINS_OPTS=--httpListenAddress=:: --httpPort=8080"
```

Or edit `/etc/default/jenkins`:
```bash
# /etc/default/jenkins
JAVA_OPTS="-Djava.net.preferIPv6Addresses=true -Djava.net.preferIPv4Stack=false"
JENKINS_ARGS="--httpListenAddress=:: --httpPort=8080"
```

Restart Jenkins:
```bash
sudo systemctl restart jenkins
```

## Step 2: Verify Jenkins Is Listening on IPv6

```bash
# Check that Jenkins is listening on IPv6
ss -6 -l -n | grep 8080
# Expected: tcp6  0  0 :::8080  :::*  LISTEN

# Or with netstat
netstat -tlnp | grep 8080
# Expected: tcp6  0  0 :::8080  :::*  LISTEN  <jenkins-pid>

# Test access via IPv6
curl -6 http://[::1]:8080/login
# Or from a remote host:
curl -6 http://[2001:db8:1:1::jenkins]:8080/login
```

## Step 3: Configure Jenkins Agent Communication via IPv6

Jenkins agents connect to the master using the JNLP (Java Network Launch Protocol) over TCP:

```groovy
// Jenkinsfile - Explicitly specify IPv6 for agent communication
pipeline {
    agent {
        label 'ipv6-agent'
    }
    environment {
        // Force Java to use IPv6 for all network connections
        JAVA_OPTS = '-Djava.net.preferIPv6Addresses=true'
    }
    stages {
        stage('Test IPv6') {
            steps {
                sh 'curl -6 https://ifconfig.me'
                sh 'ping6 -c 3 2606:4700:4700::1111'
            }
        }
    }
}
```

For agent configuration, add JVM arguments when launching the agent:

```bash
# Launch Jenkins agent with IPv6 preference
java \
    -Djava.net.preferIPv6Addresses=true \
    -Djava.net.preferIPv4Stack=false \
    -jar agent.jar \
    -url http://[2001:db8::jenkins]:8080 \
    -secret <agent-secret> \
    -name ipv6-agent \
    -webSocket
```

## Step 4: Docker Agent with IPv6

For containerized Jenkins agents, ensure the Docker network has IPv6 enabled:

```yaml
# docker-compose.yml - Jenkins with IPv6 support
version: "3.8"

services:
  jenkins:
    image: jenkins/jenkins:lts-jdk17
    container_name: jenkins
    environment:
      # Enable IPv6 in the JVM
      JAVA_OPTS: "-Djava.net.preferIPv6Addresses=true"
    ports:
      - "[::]:8080:8080"
      - "[::]:50000:50000"
    networks:
      - jenkins-net

networks:
  jenkins-net:
    enable_ipv6: true
    ipam:
      config:
        - subnet: "2001:db8:jenkins::/64"
```

## Step 5: Test IPv6 in a Jenkins Pipeline

```groovy
// Jenkinsfile - IPv6 connectivity test pipeline
pipeline {
    agent any

    stages {
        stage('IPv6 Connectivity Check') {
            steps {
                script {
                    // Check if the agent has a global IPv6 address
                    def ipv6_addrs = sh(
                        script: "ip -6 addr show scope global | grep inet6 | awk '{print \$2}'",
                        returnStdout: true
                    ).trim()
                    echo "IPv6 addresses: ${ipv6_addrs}"
                    assert ipv6_addrs != "" : "No global IPv6 address found on agent"
                }
            }
        }

        stage('IPv6 DNS Test') {
            steps {
                sh '''
                    # Test IPv6 DNS resolution
                    dig AAAA example.com +short
                    nslookup -type=AAAA google.com
                '''
            }
        }

        stage('Build with IPv6 Dependencies') {
            steps {
                sh '''
                    # Fetch dependencies over IPv6 if available
                    curl -6 -O https://example.com/dependency.tar.gz || true
                    # Fall back to IPv4 if IPv6 fails
                    curl -O https://example.com/dependency.tar.gz
                '''
            }
        }
    }
}
```

## Troubleshooting Jenkins IPv6

```bash
# If Jenkins fails to start with IPv6, check JVM IPv6 support
java -version
# Ensure you have JDK 11+ for best IPv6 support

# Check if IPv6 is available on the system
sysctl net.ipv6.conf.all.disable_ipv6
# Must be 0 (IPv6 enabled)

# Test JVM IPv6 directly
java -Djava.net.preferIPv6Addresses=true \
     -cp . TestIPv6.class
```

## Conclusion

Jenkins runs on Java, which natively supports IPv6 through JVM flags. Setting `-Djava.net.preferIPv6Addresses=true` and configuring the listener address to `::` enables Jenkins to serve both IPv4 and IPv6 clients. Agents follow the same pattern - launch with IPv6 JVM flags and connect to the master's IPv6 address. Docker-based agents require IPv6-enabled Docker networks to function correctly in dual-stack or IPv6-only environments.
