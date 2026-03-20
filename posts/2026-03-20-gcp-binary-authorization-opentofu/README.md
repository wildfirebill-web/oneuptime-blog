# How to Configure GCP Binary Authorization with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: GCP, Binary Authorization, Container Security, OpenTofu, Supply Chain, Kubernetes

Description: Learn how to configure GCP Binary Authorization policies with OpenTofu at the project and cluster level to enforce container image attestation requirements.

## Overview

GCP Binary Authorization enforces deploy-time policies requiring container images to be attested (signed) before deployment to GKE. This post focuses on the project-level Binary Authorization policy and its integration with Cloud Build for automated attestation.

## Step 1: Enable Binary Authorization

```hcl
# main.tf - Enable Binary Authorization API
resource "google_project_service" "binary_authorization" {
  service = "binaryauthorization.googleapis.com"
}

resource "google_project_service" "container_analysis" {
  service = "containeranalysis.googleapis.com"
}
```

## Step 2: Create KMS Signing Key

```hcl
# KMS keyring for attestor signing key
resource "google_kms_key_ring" "binauth_keyring" {
  name     = "binary-auth-keyring"
  location = "global"
}

resource "google_kms_crypto_key" "attestor_key" {
  name     = "ci-attestor-key"
  key_ring = google_kms_key_ring.binauth_keyring.id
  purpose  = "ASYMMETRIC_SIGN"

  version_template {
    algorithm = "EC_SIGN_P256_SHA256"
  }
}
```

## Step 3: Create Container Analysis Note and Attestor

```hcl
resource "google_container_analysis_note" "ci_build_note" {
  name = "ci-build-verified"

  attestation_authority {
    hint {
      human_readable_name = "CI/CD Build Verified"
    }
  }
}

# Attestor that CI/CD pipeline uses to sign images
resource "google_binary_authorization_attestor" "ci_attestor" {
  name = "ci-build-attestor"

  attestation_authority_note {
    note_reference = google_container_analysis_note.ci_build_note.name

    public_keys {
      id = data.google_kms_crypto_key_version.key_version.id

      pkix_public_key {
        public_key_pem      = data.google_kms_crypto_key_version.key_version.public_key[0].pem
        signature_algorithm = "ECDSA_P256_SHA256"
      }
    }
  }
}
```

## Step 4: Configure the Binary Authorization Policy

```hcl
# Project-level Binary Authorization policy
resource "google_binary_authorization_policy" "default_policy" {
  project = var.project_id

  # Default rule: require CI attestation for all images
  default_admission_rule {
    evaluation_mode  = "REQUIRE_ATTESTATION"
    enforcement_mode = "ENFORCED_BLOCK_AND_AUDIT_LOG"

    require_attestations_by = [
      google_binary_authorization_attestor.ci_attestor.name,
    ]
  }

  # Allow GKE system images without attestation
  admission_whitelist_patterns {
    name_pattern = "gcr.io/google-containers/*"
  }

  admission_whitelist_patterns {
    name_pattern = "k8s.gcr.io/*"
  }

  admission_whitelist_patterns {
    name_pattern = "gcr.io/gke-release/*"
  }

  global_policy_evaluation_mode = "ENABLE"
}
```

## Step 5: Grant Attestor Permissions to CI Service Account

```hcl
# CI/CD pipeline service account can create attestations
resource "google_container_analysis_note_iam_member" "ci_note_attacher" {
  project = var.project_id
  note    = google_container_analysis_note.ci_build_note.name
  role    = "roles/containeranalysis.notes.attacher"
  member  = "serviceAccount:${google_service_account.ci_sa.email}"
}

resource "google_binary_authorization_attestor_iam_member" "ci_signer" {
  project  = var.project_id
  attestor = google_binary_authorization_attestor.ci_attestor.name
  role     = "roles/binaryauthorization.attestorsVerifier"
  member   = "serviceAccount:${google_service_account.ci_sa.email}"
}
```

## Summary

GCP Binary Authorization with OpenTofu creates a cryptographic supply chain control that prevents unauthorized container images from running in production. The CI/CD pipeline signs images after successful builds, and the admission policy enforces that only signed images can be deployed to GKE clusters.
