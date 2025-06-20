# GitHub Actions Security Guide for Public Repositories

## Table of Contents
1. [Overview](#overview)
2. [Current Workflow Analysis](#current-workflow-analysis)
3. [What's Safe vs Unsafe in Public Workflows](#whats-safe-vs-unsafe-in-public-workflows)
4. [How Secrets Are Protected](#how-secrets-are-protected)
5. [Best Practices for Public Repository Workflows](#best-practices-for-public-repository-workflows)
6. [Required Modifications to Current Workflow](#required-modifications-to-current-workflow)
7. [Alternative Deployment Approaches](#alternative-deployment-approaches)
8. [Security Checklist](#security-checklist)

## Overview

This document provides a comprehensive security analysis of the GitHub Actions workflow for the AudioFetch application, with specific focus on security considerations for public repositories.

## Current Workflow Analysis

### Security Strengths ✅
1. **Uses GitHub's built-in GITHUB_TOKEN** for container registry authentication
2. **Leverages GitHub Secrets** for sensitive information
3. **Minimal permissions** granted to jobs (contents: read, packages: write)
4. **Uses trusted official actions** from GitHub and Docker

### Security Concerns ⚠️
1. **Secrets exposed in deployment script** - The deployment job writes secrets directly into files on the VPS
2. **Plain text .env file creation** - Sensitive configuration is written unencrypted
3. **Docker compose file generation** - Repository name is exposed in the deployment script
4. **No audit logging** for deployment actions
5. **Direct SSH access** to production server from GitHub Actions

## What's Safe vs Unsafe in Public Workflows

### ✅ SAFE Practices
- Using `${{ secrets.* }}` for sensitive values
- Building and pushing Docker images to GitHub Container Registry
- Using `GITHUB_TOKEN` for registry authentication
- Metadata extraction and tagging
- Running tests and linting
- Building artifacts

### ❌ UNSAFE Practices
- Hardcoding sensitive values in workflow files
- Echoing secrets to logs (even accidentally)
- Creating files with embedded secrets (current workflow does this)
- Using third-party actions without version pinning
- Granting excessive permissions to workflow jobs
- Storing production configuration in the repository

## How Secrets Are Protected

### GitHub's Secret Protection
1. **Encryption at rest** - All secrets are encrypted in GitHub's database
2. **Masked in logs** - GitHub automatically masks secret values in workflow logs
3. **Access control** - Only users with write access can manage secrets
4. **No API access** - Secrets cannot be retrieved via API, only used in workflows

### Current Workflow Secret Usage
The workflow uses the following secrets:
- `VPS_HOST` - Server hostname/IP
- `VPS_USERNAME` - SSH username
- `VPS_SSH_KEY` - Private SSH key for authentication
- `VPS_PORT` - SSH port
- `VPS_APP_DIR` - Application directory path
- `APP_PORT` - Application port
- `LOG_LEVEL` - Logging verbosity
- `DOWNLOADS_HOST_PATH` - Host path for downloads
- `GITHUB_TOKEN` - Automatically provided by GitHub

### ⚠️ Security Risk
The current workflow writes these secrets into plain text files on the VPS, which could be compromised if the server is breached.

## Best Practices for Public Repository Workflows

### 1. Principle of Least Privilege
```yaml
permissions:
  contents: read       # Only what's needed
  packages: write      # Only for jobs that push images
  actions: read        # If reading workflow artifacts
```

### 2. Version Pin All Actions
```yaml
# Good
uses: actions/checkout@v4.1.1

# Bad
uses: actions/checkout@main
```

### 3. Use Environment Protection Rules
```yaml
environment:
  name: production
  url: https://your-app.com
```

### 4. Implement Deployment Approvals
Configure environment protection rules in GitHub settings to require manual approval for production deployments.

### 5. Audit and Monitor
- Enable GitHub Advanced Security features
- Review workflow runs regularly
- Monitor for suspicious activity

### 6. Separate Configuration from Code
Store production configuration separately from the repository, not generated by CI/CD.

## Required Modifications to Current Workflow

### 1. Remove Direct Secret Injection
Instead of creating .env files with secrets, use one of these approaches:

#### Option A: Pre-configured Environment
```yaml
- name: Deploy to VPS
  uses: appleboy/ssh-action@v1.0.0
  with:
    host: ${{ secrets.VPS_HOST }}
    username: ${{ secrets.VPS_USERNAME }}
    key: ${{ secrets.VPS_SSH_KEY }}
    port: ${{ secrets.VPS_PORT }}
    script: |
      cd ${{ secrets.VPS_APP_DIR }}
      # Assume .env already exists on server with proper permissions
      docker compose pull
      docker compose up -d
      docker image prune -f
```

#### Option B: Encrypted Configuration
```yaml
- name: Deploy encrypted config
  uses: appleboy/ssh-action@v1.0.0
  with:
    host: ${{ secrets.VPS_HOST }}
    username: ${{ secrets.VPS_USERNAME }}
    key: ${{ secrets.VPS_SSH_KEY }}
    port: ${{ secrets.VPS_PORT }}
    script: |
      cd ${{ secrets.VPS_APP_DIR }}
      # Decrypt pre-uploaded encrypted config
      gpg --decrypt --output .env .env.gpg
      docker compose pull
      docker compose up -d
      docker image prune -f
```

### 2. Use Docker Secrets or Environment Variables
Modify docker-compose.yml to use Docker secrets:

```yaml
services:
  web:
    image: ghcr.io/username/repo:latest
    secrets:
      - admin_password
      - secret_key
    environment:
      - PORT=${PORT}
      - HOST=${HOST}

secrets:
  admin_password:
    file: ./secrets/admin_password.txt
  secret_key:
    file: ./secrets/secret_key.txt
```

### 3. Implement Secure Configuration Management
Create a separate configuration repository or use a secrets management service:

```yaml
- name: Fetch configuration from secure store
  run: |
    # Example using HashiCorp Vault, AWS Secrets Manager, etc.
    vault kv get -field=env secret/myapp > .env.encrypted
```

### 4. Add Security Scanning
```yaml
- name: Run security scan
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ steps.meta.outputs.version }}
    format: 'sarif'
    output: 'trivy-results.sarif'

- name: Upload Trivy scan results
  uses: github/codeql-action/upload-sarif@v2
  with:
    sarif_file: 'trivy-results.sarif'
```

## Alternative Deployment Approaches

### 1. GitOps with ArgoCD/Flux
- Repository contains only application code
- Separate repository for Kubernetes manifests
- Automated sync with cluster
- Secrets managed by Kubernetes

### 2. Cloud Provider Integration
- Use GitHub Actions with cloud provider CLI
- Deploy to managed services (ECS, Cloud Run, App Service)
- Leverage cloud-native secret management

### 3. Self-Hosted Runners
- Run GitHub Actions runners in your own infrastructure
- Better control over secret management
- Reduced attack surface

### 4. Webhook-Based Deployment
```yaml
- name: Trigger deployment
  run: |
    curl -X POST ${{ secrets.DEPLOYMENT_WEBHOOK }} \
      -H "Authorization: Bearer ${{ secrets.DEPLOYMENT_TOKEN }}" \
      -d '{"image": "ghcr.io/${{ github.repository }}:latest"}'
```

## Security Checklist

### Pre-Deployment
- [ ] All secrets are stored in GitHub Secrets
- [ ] No sensitive data in workflow files
- [ ] Actions are version-pinned
- [ ] Minimal permissions granted
- [ ] Security scanning enabled

### Deployment
- [ ] Secrets are not written to plain text files
- [ ] Configuration is encrypted or pre-existing
- [ ] Audit logs are enabled
- [ ] Deployment requires approval (for production)

### Post-Deployment
- [ ] Old images are cleaned up
- [ ] Deployment logs are reviewed
- [ ] No secrets in container logs
- [ ] Server configuration is secure

## Recommended Immediate Actions

1. **CRITICAL**: Remove the `.env` file generation from the workflow
2. **HIGH**: Pre-configure the VPS with encrypted environment files
3. **HIGH**: Implement deployment approvals for production
4. **MEDIUM**: Add container security scanning
5. **MEDIUM**: Version pin all GitHub Actions
6. **LOW**: Add deployment notifications/monitoring

## Example Secure Workflow

Here's a modified version of your workflow with security improvements:

```yaml
name: Secure Deploy to VPS

on:
  push:
    branches:
      - main

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
      security-events: write  # For security scanning

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4.1.1

      - name: Log in to the Container registry
        uses: docker/login-action@v3.0.0
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5.0.0
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=semver,pattern={{version}}
            type=sha,prefix={{branch}}-

      - name: Build and push Docker image
        uses: docker/build-push-action@v5.0.0
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}

      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@0.16.1
        with:
          image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ steps.meta.outputs.version }}
          format: 'sarif'
          output: 'trivy-results.sarif'

      - name: Upload Trivy scan results
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: 'trivy-results.sarif'

  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://your-app-url.com

    steps:
      - name: Deploy to VPS
        uses: appleboy/ssh-action@v1.0.0
        with:
          host: ${{ secrets.VPS_HOST }}
          username: ${{ secrets.VPS_USERNAME }}
          key: ${{ secrets.VPS_SSH_KEY }}
          port: ${{ secrets.VPS_PORT }}
          script: |
            cd ${{ secrets.VPS_APP_DIR }}
            
            # Verify pre-existing encrypted configuration
            if [ ! -f .env ]; then
              echo "ERROR: .env file not found. Please configure server manually."
              exit 1
            fi
            
            # Log deployment (without sensitive info)
            echo "$(date): Deploying image ghcr.io/${{ github.repository }}:latest" >> deployment.log
            
            # Login to registry
            echo "${{ secrets.GITHUB_TOKEN }}" | docker login ghcr.io -u ${{ github.actor }} --password-stdin
            
            # Deploy using pre-existing configuration
            docker compose pull
            docker compose down
            docker compose up -d
            
            # Cleanup
            docker image prune -f
            docker logout ghcr.io
            
            # Verify deployment
            docker compose ps
```

## Conclusion

While GitHub Actions is powerful for CI/CD in public repositories, special care must be taken to ensure secrets and sensitive configuration are never exposed. The current workflow has significant security risks that should be addressed immediately by removing direct secret injection and implementing proper configuration management on the target server.