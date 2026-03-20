# How to Create Custom Templates from the Web Editor in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Templates, Web Editor, DevOps

Description: Learn how to create and save custom Portainer templates directly from the browser-based web editor.

## Introduction

The Portainer web editor is the quickest way to create custom templates. You write or paste a Docker Compose file directly in the browser, add template variables, and save it to your custom template catalog. This guide covers the complete process of creating templates via the web editor.

## Prerequisites

- Portainer CE or BE installed
- Admin or environment-admin access
- A Docker Compose file you want to templatize

## Step 1: Open the Custom Template Creator

1. Log in to Portainer
2. Select your Docker environment
3. Click **App Templates** in the left sidebar
4. Click the **Custom templates** tab
5. Click **+ Add custom template**
6. Select **Web editor** as the method

## Step 2: Fill in Template Information

Complete the metadata section:

```text
Title:       Monitoring Stack
Description: Prometheus and Grafana monitoring with alerting

Categories:  monitoring, observability
Platform:    linux
Type:        Stack
Logo URL:    https://raw.githubusercontent.com/grafana/grafana/main/public/img/grafana_icon.svg
```

## Step 3: Write the Compose File in the Editor

The web editor provides syntax highlighting and basic validation. Enter your Compose file:

```yaml
version: "3.8"

services:
  prometheus:
    image: prom/prometheus:{{ .prometheus_version | default "v2.48.0" }}
    ports:
      - "{{ .prometheus_port | default "9090" }}:9090"
    volumes:
      - prometheus-config:/etc/prometheus
      - prometheus-data:/prometheus
    command:
      - "--config.file=/etc/prometheus/prometheus.yml"
      - "--storage.tsdb.path=/prometheus"
      - "--storage.tsdb.retention.time={{ .retention | default "30d" }}"
      - "--web.enable-lifecycle"
    restart: unless-stopped
    networks:
      - monitoring

  grafana:
    image: grafana/grafana:{{ .grafana_version | default "10.2.0" }}
    ports:
      - "{{ .grafana_port | default "3000" }}:3000"
    environment:
      - GF_SECURITY_ADMIN_USER={{ .admin_user | default "admin" }}
      - GF_SECURITY_ADMIN_PASSWORD={{ .admin_password }}
      - GF_USERS_ALLOW_SIGN_UP=false
    volumes:
      - grafana-data:/var/lib/grafana
    depends_on:
      - prometheus
    restart: unless-stopped
    networks:
      - monitoring

  alertmanager:
    image: prom/alertmanager:latest
    ports:
      - "{{ .alertmanager_port | default "9093" }}:9093"
    volumes:
      - alertmanager-data:/alertmanager
    restart: unless-stopped
    networks:
      - monitoring

volumes:
  prometheus-config:
  prometheus-data:
  grafana-data:
  alertmanager-data:

networks:
  monitoring:
    driver: bridge
```

## Step 4: Add Variables to the Template

Scroll down to the **Variables** section. Add entries for each variable used in the Compose file:

### Variable: admin_password (Required)

```json
{
  "name": "admin_password",
  "label": "Grafana admin password",
  "description": "Password for the Grafana admin user",
  "default": ""
}
```

### Variable: prometheus_port (Optional)

```json
{
  "name": "prometheus_port",
  "label": "Prometheus port",
  "description": "Host port for Prometheus UI",
  "default": "9090"
}
```

### Variable: grafana_port (Optional)

```json
{
  "name": "grafana_port",
  "label": "Grafana port",
  "description": "Host port for Grafana UI",
  "default": "3000"
}
```

In the Portainer UI, click **Add variable** for each entry and fill in the name, label, description, and default value.

## Step 5: Use Mustache Variable Syntax

Portainer uses Mustache-style syntax for template variables:

```yaml
# Basic variable substitution

image: myapp:{{ .version }}

# Variable with default value (uses pipe | default)
port: {{ .port | default "8080" }}

# Variable used in a string
environment:
  - APP_URL=https://{{ .domain }}.example.com

# Variable in a volume path
volumes:
  - {{ .data_dir | default "/data" }}:/app/data
```

**Note:** The `.` (dot) before variable names is required.

## Step 6: Preview the Resolved Template

Before saving, you can mentally verify by substituting test values into your variables to ensure the Compose file would be valid YAML.

For example, with `admin_password=secret123` and `grafana_port=3001`:

```yaml
# Expected resolved output (for verification)
grafana:
  ports:
    - "3001:3000"
  environment:
    - GF_SECURITY_ADMIN_PASSWORD=secret123
```

## Step 7: Save the Template

1. Review all fields and the Compose file
2. Click **Create custom template**
3. The template is saved and appears in the **Custom templates** catalog

## Step 8: Test the Template

1. Click on your newly created template
2. The variable fields appear with their labels and defaults
3. Fill in the required values (those without defaults)
4. Click **Deploy the stack**
5. Verify the stack deploys successfully

## Editing an Existing Template

1. Go to **App Templates > Custom templates**
2. Click the **Edit** (pencil) icon on your template
3. Modify the Compose file or variables
4. Save changes

## Tips for Web Editor Templates

- Use `default` filters liberally to make templates user-friendly
- Test your Compose file without variables first, then add parameterization
- Keep variable names short and descriptive
- Group related variables logically in the template form

## Conclusion

The Portainer web editor makes it easy to create custom templates directly in the browser. For quick templates and iteration, it is the fastest approach. As your templates mature, consider moving them to a Git repository for version control and team collaboration.
