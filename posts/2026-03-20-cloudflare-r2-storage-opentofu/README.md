# How to Set Up Cloudflare R2 Storage with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Cloudflare, R2, Object Storage, Infrastructure as Code, CDN, Storage

Description: Learn how to create and configure Cloudflare R2 storage buckets using OpenTofu for zero-egress-cost object storage integrated with Cloudflare's global network.

---

Cloudflare R2 is S3-compatible object storage with no egress fees, making it ideal for serving large files globally. OpenTofu's Cloudflare provider lets you manage R2 buckets, CORS rules, and custom domain bindings as reproducible infrastructure code.

## Provider Configuration

```hcl
# main.tf
terraform {
  required_providers {
    cloudflare = {
      source  = "cloudflare/cloudflare"
      version = "~> 4.23"
    }
  }
}

provider "cloudflare" {
  api_token = var.cloudflare_api_token
}
```

## Creating an R2 Bucket

```hcl
# bucket.tf
# Create an R2 bucket for media storage
resource "cloudflare_r2_bucket" "media" {
  account_id = var.cloudflare_account_id
  name       = "${var.project_name}-media-${var.environment}"
  location   = "WNAM"  # Western North America — EEUR, ENAM, APAC, WEUR, WNAM, OC

  lifecycle {
    # Prevent accidental bucket deletion
    prevent_destroy = true
  }
}

# Create a bucket for static site assets
resource "cloudflare_r2_bucket" "static" {
  account_id = var.cloudflare_account_id
  name       = "${var.project_name}-static-${var.environment}"
  location   = "WNAM"
}
```

## Setting Up a Custom Domain with R2

```hcl
# custom_domain.tf
# Create a Worker to serve R2 content on a custom domain
resource "cloudflare_worker_script" "r2_server" {
  account_id = var.cloudflare_account_id
  name       = "r2-media-server"

  content = <<-JS
    export default {
      async fetch(request, env) {
        const url = new URL(request.url);
        const key = url.pathname.slice(1); // Remove leading /

        if (!key) {
          return new Response('Not Found', { status: 404 });
        }

        const object = await env.R2_BUCKET.get(key);

        if (!object) {
          return new Response('Not Found', { status: 404 });
        }

        const headers = new Headers();
        object.writeHttpMetadata(headers);
        headers.set('etag', object.httpEtag);
        // Cache for 1 year for immutable content
        headers.set('cache-control', 'public, max-age=31536000, immutable');

        return new Response(object.body, { headers });
      },
    };
  JS

  r2_bucket_binding {
    name        = "R2_BUCKET"
    bucket_name = cloudflare_r2_bucket.media.name
  }
}

# Route the custom domain to the Worker
resource "cloudflare_worker_route" "r2_route" {
  zone_id     = var.cloudflare_zone_id
  pattern     = "media.${var.domain_name}/*"
  script_name = cloudflare_worker_script.r2_server.name
}
```

## Creating API Tokens for Application Access

```hcl
# api_tokens.tf
# Create an API token scoped to R2 operations
resource "cloudflare_api_token" "r2_access" {
  name = "r2-application-token"

  policy {
    permission_groups = [
      data.cloudflare_api_token_permission_groups.all.object_read_and_write["Workers R2 Storage:Edit"],
    ]

    resources = {
      "com.cloudflare.api.account.${var.cloudflare_account_id}" = "*"
    }
  }
}

data "cloudflare_api_token_permission_groups" "all" {}

output "r2_api_token" {
  value     = cloudflare_api_token.r2_access.value
  sensitive = true
}
```

## Best Practices

- Use Workers to serve R2 content on custom domains — R2 doesn't have built-in public URL support without Workers or the R2 public bucket feature.
- Set `Cache-Control: immutable` headers for content-addressed (hash-named) assets to maximize CDN caching.
- Use location hints when creating buckets to place storage closer to your primary user base.
- Create scoped API tokens for each application rather than using your account-level API token.
- Enable lifecycle rules once Cloudflare R2 supports them — monitor bucket growth to control storage costs.
