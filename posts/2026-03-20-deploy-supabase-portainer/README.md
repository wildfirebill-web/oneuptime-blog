# How to Deploy Supabase via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Supabase, PostgreSQL, Docker, Backend as a Service

Description: Deploy Supabase open-source Firebase alternative using Portainer for a self-hosted backend with PostgreSQL, authentication, and real-time APIs.

## Introduction

Supabase is an open-source Firebase alternative that provides a PostgreSQL database, authentication, instant REST and GraphQL APIs, real-time subscriptions, edge functions, and file storage. This guide deploys Supabase self-hosted using the official Docker Compose configuration.

## Prerequisites

- Portainer installed with Docker
- At least 2 GB RAM
- A domain name for the `SITE_URL` and JWT configuration

## Step 1: Clone the Supabase Self-Hosted Repository

```bash
# Clone the official self-hosted compose files

git clone --depth 1 https://github.com/supabase/supabase
cd supabase/docker

# Copy the example env file
cp .env.example .env
```

## Step 2: Generate Required Secrets

```bash
# Generate JWT secret
openssl rand -base64 32

# Generate ANON_KEY and SERVICE_ROLE_KEY (JWTs signed with the JWT_SECRET)
# Use the Supabase JWT generator: https://supabase.com/docs/guides/self-hosting/docker#generate-api-keys
```

## Step 3: Configure Environment Variables

Edit `.env` with your values:

```bash
# Site URL
SITE_URL=http://your-domain.com
ADDITIONAL_REDIRECT_URLS=

# API
API_EXTERNAL_URL=http://your-domain.com

# JWT - generate with the Supabase JWT tool
JWT_SECRET=your-super-secret-jwt-token-with-at-least-32-characters-long
ANON_KEY=your-generated-anon-key
SERVICE_ROLE_KEY=your-generated-service-role-key

# Database
POSTGRES_PASSWORD=your-super-secret-postgres-password

# Dashboard
DASHBOARD_USERNAME=supabase
DASHBOARD_PASSWORD=your-dashboard-password

# SMTP
SMTP_ADMIN_EMAIL=admin@your-domain.com
SMTP_HOST=smtp.your-domain.com
SMTP_PORT=587
SMTP_USER=your-smtp-user
SMTP_PASS=your-smtp-password
SMTP_SENDER_NAME=Supabase
```

## Step 4: Deploy via Portainer

In Portainer, navigate to **Stacks** > **Add Stack** > **Upload**, and upload the `docker-compose.yml` from the `supabase/docker` directory, along with the volumes directory.

Alternatively, deploy from the Git repository:
1. Select **Repository** as the build method
2. Enter `https://github.com/supabase/supabase`
3. Set the compose file path to `docker/docker-compose.yml`

## Step 5: Access the Supabase Studio

Open `http://<host>:3000` and log in with your `DASHBOARD_USERNAME` and `DASHBOARD_PASSWORD`.

## Step 6: Use the Supabase Client SDK

```javascript
// npm install @supabase/supabase-js
import { createClient } from '@supabase/supabase-js'

const supabase = createClient(
  'http://your-domain.com',
  'your-anon-key'
)

// Query data
const { data, error } = await supabase
  .from('users')
  .select('*')
  .limit(10)

// Insert a row
const { data: newUser, error: insertError } = await supabase
  .from('users')
  .insert({ name: 'Alice', email: 'alice@example.com' })
  .select()
  .single()

// Authenticate
const { data: session, error: authError } = await supabase.auth.signInWithPassword({
  email: 'user@example.com',
  password: 'password'
})
```

## Conclusion

Supabase self-hosted deploys ~15 containers (Kong API gateway, PostgREST, GoTrue auth, Realtime, Storage, Analytics, PostgreSQL). The `ANON_KEY` is safe for client-side use and respects Row Level Security (RLS) policies. The `SERVICE_ROLE_KEY` bypasses RLS - keep it server-side only. Always enable RLS on tables before exposing them via the `ANON_KEY`.
