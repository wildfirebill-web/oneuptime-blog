# How to Set Up a Docker Learning Lab with Portainer - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Learning, Education, Lab

Description: Build an interactive Docker learning lab using Portainer where students can experiment with containers safely in isolated environments.

## Introduction

A Docker learning lab provides hands-on experience with containers without the risk of breaking production systems. Portainer's team-based access control makes it ideal for educational settings: each student gets their own isolated environment, instructors have oversight, and the visual interface lowers the barrier to entry. This guide sets up a complete learning lab infrastructure.

## Architecture

The learning lab consists of:
- A Docker host (or small cluster) with Portainer installed
- One Portainer team per student or lab group
- Namespace or network isolation between teams
- Pre-built exercise stacks for each lesson

## Step 1: Install Portainer for the Lab

```bash
# Install Docker on the lab server

curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER

# Deploy Portainer
docker volume create portainer_data
docker run -d \
  --name portainer \
  --restart=always \
  -p 8000:8000 \
  -p 9443:9443 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest

echo "Portainer running at https://localhost:9443"
```

## Step 2: Create Student Accounts via API

```bash
#!/bin/bash
# create-student-accounts.sh
PORTAINER_URL="https://portainer.lab.local"
ADMIN_TOKEN=$(curl -s -X POST \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"admin-password"}' \
  "$PORTAINER_URL/api/auth" | python3 -c "import sys,json; print(json.load(sys.stdin)['jwt'])")

# Read student list from CSV: username,email
while IFS=',' read -r username email; do
  # Create user account
  PASSWORD="Lab$(echo $RANDOM | md5sum | head -c8)"
  
  curl -s -X POST \
    -H "Authorization: Bearer $ADMIN_TOKEN" \
    -H "Content-Type: application/json" \
    -d "{\"username\":\"$username\",\"password\":\"$PASSWORD\",\"role\":2}" \
    "$PORTAINER_URL/api/users"
  
  echo "$username,$PASSWORD" >> student-credentials.csv
  echo "Created account for: $username"
done < students.csv

echo "Credentials saved to student-credentials.csv"
```

## Step 3: Set Up Lab Exercise Stacks

Create pre-built exercises that students deploy:

```yaml
# exercise-01-hello-nginx/docker-compose.yml
version: '3.8'
services:
  web:
    image: nginx:alpine
    ports:
      - "8080:80"
    volumes:
      - ./html:/usr/share/nginx/html:ro
    labels:
      lab.exercise: "01"
      lab.topic: "nginx-basics"
```

```yaml
# exercise-02-multi-container/docker-compose.yml
version: '3.8'
services:
  wordpress:
    image: wordpress:latest
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_USER: wp_user
      WORDPRESS_DB_PASSWORD: wp_pass
      WORDPRESS_DB_NAME: wordpress
    ports:
      - "8081:80"
    depends_on:
      - db

  db:
    image: mysql:8.0
    environment:
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wp_user
      MYSQL_PASSWORD: wp_pass
      MYSQL_ROOT_PASSWORD: root_pass
    volumes:
      - db_data:/var/lib/mysql

volumes:
  db_data:
```

## Step 4: Configure Resource Limits Per Student

Prevent any student from monopolizing lab resources:

```bash
# Create an overlay network per student team
for student in $(cut -d',' -f1 students.csv); do
  docker network create --driver overlay \
    --subnet "10.${RANDOM:0:2}.0.0/24" \
    "lab-${student}-net" 2>/dev/null || true
done
```

When using Portainer BE, set resource quotas per team in **Settings > Teams > [Team] > Resource Quota**.

## Step 5: Create a Lab Exercises Repository

```bash
# Structure for lab exercises
lab-exercises/
├── 01-run-first-container/
│   ├── README.md          # Instructions
│   ├── docker-compose.yml # Exercise stack
│   └── solution/          # Solution for instructors
├── 02-volumes-and-data/
├── 03-custom-images/
├── 04-multi-service-apps/
├── 05-environment-variables/
└── 06-networking-basics/
```

```bash
# Host exercises in a Git repo and configure Portainer
# Stacks > Add Stack > Git Repository
# Repository URL: https://github.com/your-org/lab-exercises
# Compose path: 01-run-first-container/docker-compose.yml
```

## Step 6: Instructor Dashboard

Create a monitoring view for the instructor:

```bash
#!/bin/bash
# instructor-dashboard.sh - Show all student containers
PORTAINER_URL="https://portainer.lab.local"
API_KEY="instructor-api-key"

echo "=== Student Container Status ==="
curl -s \
  -H "X-API-Key: $API_KEY" \
  "$PORTAINER_URL/api/endpoints/1/docker/containers/json?all=true" | \
  python3 -c "
import sys, json
containers = json.load(sys.stdin)
for c in containers:
  labels = c.get('Labels', {})
  exercise = labels.get('lab.exercise', 'unknown')
  name = c['Names'][0] if c['Names'] else 'unnamed'
  status = c['Status']
  print(f'Exercise {exercise}: {name} - {status}')
"
```

## Step 7: Auto-Cleanup Script

Clean up student environments after each lab session:

```bash
#!/bin/bash
# cleanup-lab.sh - Remove all lab containers and volumes
echo "Cleaning up lab environment..."

# Stop and remove containers with lab labels
docker ps -a --filter "label=lab.exercise" --format "{{.ID}}" | \
  xargs -r docker rm -f

# Remove lab volumes
docker volume ls --filter "label=lab.exercise" --format "{{.Name}}" | \
  xargs -r docker volume rm

# Remove lab networks
docker network ls --filter "name=lab-" --format "{{.Name}}" | \
  grep -v "portainer" | \
  xargs -r docker network rm

echo "Lab cleanup complete"
```

## Conclusion

A Portainer-based Docker learning lab provides students with isolated, visual container environments that reduce friction in learning Docker concepts. The team-based access control keeps students separated, while the API enables automated provisioning and cleanup between sessions. Pre-built exercise stacks let instructors focus on teaching rather than environment setup, making Portainer an ideal platform for Docker education at any scale.
