# docker-credential-helpers

A bunch of Docker credential helpers.

Currently supports:
- GitLab (for CI agent)
- Azure Container Registry (via azure-cli)

## Usage

```shell
cat > /root/.docker/config.json <<EOF
{
  "credHelpers": {
    "registry.gitlab.com": "gitlab",
    "your-registry.azurecr.io": "acr"
  }
}
EOF
```
