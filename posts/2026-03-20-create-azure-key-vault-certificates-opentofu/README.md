# How to Create Azure Key Vault Certificates with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Azure, Key Vault, Certificate, Infrastructure as Code

Description: Learn how to create and manage TLS certificates in Azure Key Vault with OpenTofu, including self-signed certificates and certificates from integrated CAs.

Azure Key Vault manages the full lifecycle of TLS certificates including generation, renewal, and deployment. Managing certificates in OpenTofu ensures consistent issuance policies, auto-renewal settings, and access control across services.

## Self-Signed Certificate

```hcl
resource "azurerm_key_vault_certificate" "self_signed" {
  name         = "internal-tls"
  key_vault_id = azurerm_key_vault.main.id

  certificate_policy {
    issuer_parameters {
      name = "Self"  # Self-signed
    }

    key_properties {
      exportable = true
      key_size   = 2048
      key_type   = "RSA"
      reuse_key  = true
    }

    lifetime_action {
      action {
        action_type = "AutoRenew"
      }

      trigger {
        days_before_expiry = 30  # Renew 30 days before expiry
      }
    }

    secret_properties {
      content_type = "application/x-pkcs12"
    }

    x509_certificate_properties {
      extended_key_usage = ["1.3.6.1.5.5.7.3.1"]  # Server Authentication
      key_usage          = ["digitalSignature", "keyEncipherment"]

      subject            = "CN=internal.example.com"
      validity_in_months = 12

      subject_alternative_names {
        dns_names = ["internal.example.com", "*.internal.example.com"]
      }
    }
  }
}
```

## Certificate from DigiCert (Integrated CA)

```hcl
# First, add DigiCert as a certificate issuer

resource "azurerm_key_vault_certificate_issuer" "digicert" {
  name          = "DigiCert"
  key_vault_id  = azurerm_key_vault.main.id
  provider_name = "DigiCert"

  account_id = var.digicert_account_id
  password   = var.digicert_api_key

  admin {
    email_address = "pki@example.com"
    first_name    = "PKI"
    last_name     = "Admin"
    phone         = "+1-555-0100"
  }
}

resource "azurerm_key_vault_certificate" "public_tls" {
  name         = "example-com-tls"
  key_vault_id = azurerm_key_vault.main.id

  certificate_policy {
    issuer_parameters {
      name = azurerm_key_vault_certificate_issuer.digicert.name
    }

    key_properties {
      exportable = true
      key_size   = 2048
      key_type   = "RSA"
      reuse_key  = false
    }

    lifetime_action {
      action {
        action_type = "AutoRenew"
      }
      trigger {
        days_before_expiry = 30
      }
    }

    secret_properties {
      content_type = "application/x-pkcs12"
    }

    x509_certificate_properties {
      extended_key_usage = ["1.3.6.1.5.5.7.3.1"]
      key_usage          = ["digitalSignature", "keyEncipherment"]
      subject            = "CN=example.com"
      validity_in_months = 12

      subject_alternative_names {
        dns_names = ["example.com", "www.example.com", "api.example.com"]
      }
    }
  }

  depends_on = [azurerm_key_vault_certificate_issuer.digicert]
}
```

## Import Existing Certificate

```hcl
resource "azurerm_key_vault_certificate" "imported" {
  name         = "imported-cert"
  key_vault_id = azurerm_key_vault.main.id

  certificate {
    contents = filebase64("cert.pfx")
    password = var.pfx_password
  }
}
```

## RBAC for Certificate Access

```hcl
# Allow App Service to read certificate
resource "azurerm_role_assignment" "app_cert_reader" {
  scope                = azurerm_key_vault.main.id
  role_definition_name = "Key Vault Certificate User"
  principal_id         = azurerm_app_service.main.identity[0].principal_id
}

# Allow Application Gateway to read certificate
resource "azurerm_role_assignment" "agw_cert_reader" {
  scope                = azurerm_key_vault.main.id
  role_definition_name = "Key Vault Certificate User"
  principal_id         = azurerm_application_gateway.main.identity[0].principal_id
}
```

## Outputs

```hcl
output "certificate_secret_id" {
  description = "Secret ID for the certificate - used by App Gateway and App Service"
  value       = azurerm_key_vault_certificate.self_signed.secret_id
}

output "certificate_thumbprint" {
  value = azurerm_key_vault_certificate.self_signed.thumbprint
}
```

## Conclusion

Azure Key Vault certificates in OpenTofu automate the full TLS lifecycle. Configure auto-renewal to rotate certificates 30 days before expiry, integrate with DigiCert or other CAs for trusted public certificates, and use RBAC to grant Key Vault Certificate User access to services that consume the certificate. Reference the certificate's secret_id in Application Gateway and App Service to enable automatic certificate deployment.
