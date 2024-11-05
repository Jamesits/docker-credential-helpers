# docker-credential-helpers

A bunch of Docker credential helpers.

Currently supports:
- GitLab (for CI agent)
- Azure Container Registry (via azure-cli)

## Usage

### GitLab

TBD.

### Azure Container Registry

Configure your `az login` args in `/etc/docker-credential-helpers/acr/<your-registry>`, then add the helper to your Docker client config.

Example: using certificate authentication

Create the certificate ([EC certs not supported](https://github.com/Azure/azure-cli/issues/30254)):
```shell
openssl genrsa -out device.key 4096
openssl req -new -x509 -key device.key -out device.crt -days 720

cat device.key device.crt > /etc/ssl/private/device.pem
```

Create the service principal in your ~~Azure AD~~ Entra ID:
```hcl
resource "azuread_application" "device" {
  display_name = "device"
  prevent_duplicate_names = true
}

resource "azuread_service_principal" "device" {
  client_id = azuread_application.device.client_id
  tags = [
    "AppServiceIntegratedApp",
    "WindowsAzureActiveDirectoryIntegratedApp",
    "HideApp",
  ]
  app_role_assignment_required = true
}

# Note: this entry expires every ~2 years and need to be re-created by then.
resource "azuread_application_certificate" "device" {
  application_id = azuread_application.device.id
  type           = "AsymmetricX509Cert"
  value          = file("device.crt")
}
```

Set your login config:
```shell
cat > /etc/docker-credential-helpers/acr/your-registry <<EOF
# shellcheck shell=bash
AZ_LOGIN_OPTS=(
    "--service-principal"
    "--tenant" "00000000-0000-0000-0000-000000000000" # Tenant ID
    "--username" "00000000-0000-0000-0000-000000000000" # Client ID
    "--password" "/etc/ssl/private/device.pem" # Path to the client certificate
)
EOF
```

Set your Docker client config:
```shell
cat > /root/.docker/config.json <<EOF
{
  "credHelpers": {
    "your-registry.azurecr.io": "acr"
  }
}
EOF
```

## Notes

### GitLab CI Runner

Bugs:
- It always tries to use the first `credHelpers` entry regardless of the registry.
- Runner installed via a package manager always tries to load `/root/.docker/config.json` regardless of its user.

For Docker-in-Docker to work with this kind of configuration, you need to mount everything into the container by default, modify all containers to install `azure-cli` and try not to break others running `docker login` inside the CI script. Good luck.
