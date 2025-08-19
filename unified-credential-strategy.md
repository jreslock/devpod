# Unified Credential Strategy for OBT

## Overview

This document outlines a unified credential strategy that works across:
- **DevPod + AWS Provider** (remote EC2 instances)
- **DevPod + Local Docker Provider** (local development)
- **GitHub Actions** (using `devcontainer up` and `devcontainer exec`)

The strategy uses environment variables as the common foundation, with each platform providing those variables through its native mechanisms.

## Core Principle: Environment Variable Contract

Define a standard set of environment variables that the OBT devcontainer expects:

```bash
# AWS Credentials
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
AWS_SESSION_TOKEN          # Optional for temporary credentials
AWS_DEFAULT_REGION

# GitHub Authentication
GITHUB_TOKEN

# Docker/Container Registry
DOCKER_CONFIG_JSON         # Base64 encoded Docker config.json content

# Additional Optional Credentials
ECR_REGISTRY_URL          # If using specific ECR registry
GPG_PRIVATE_KEY           # If GPG signing is needed
SSH_PRIVATE_KEY           # If SSH access is needed
```

## Implementation Strategy by Environment

### 1. DevPod with AWS Provider

**Credential Source**: DevPod's credential injection + AWS Instance Profiles

```bash
# Configure DevPod credential injection
devpod context set-options default \
  -o SSH_INJECT_GIT_CREDENTIALS=true \
  -o SSH_INJECT_DOCKER_CREDENTIALS=true

# Use AWS Instance Profile for AWS credentials (recommended)
devpod provider set-options aws \
  --option AWS_INSTANCE_PROFILE_ARN=arn:aws:iam::ACCOUNT:instance-profile/ObtDevPodRole

# Alternative: Set AWS credentials in provider (less secure)
devpod provider set-options aws \
  --option AWS_ACCESS_KEY_ID="${AWS_ACCESS_KEY_ID}" \
  --option AWS_SECRET_ACCESS_KEY="${AWS_SECRET_ACCESS_KEY}"
```

**In devcontainer.json:**
```json
{
  "name": "OBT DevContainer",
  "dockerComposeFile": "docker-compose.yml", 
  "service": "obt",
  "containerEnv": {
    "AWS_DEFAULT_REGION": "us-west-2",
    "GITHUB_TOKEN": "${localEnv:GITHUB_TOKEN}"
  },
  "postCreateCommand": ".devcontainer/setup-credentials.sh"
}
```

### 2. DevPod with Local Docker Provider

**Credential Source**: Local environment variables + DevPod credential injection

```bash
# Enable credential injection
devpod context set-options default \
  -o SSH_INJECT_GIT_CREDENTIALS=true \
  -o SSH_INJECT_DOCKER_CREDENTIALS=true

# Set local environment variables
export AWS_ACCESS_KEY_ID="your-key"
export AWS_SECRET_ACCESS_KEY="your-secret" 
export GITHUB_TOKEN="ghp_your-token"
```

**In devcontainer.json** (same as AWS):
```json
{
  "containerEnv": {
    "AWS_ACCESS_KEY_ID": "${localEnv:AWS_ACCESS_KEY_ID}",
    "AWS_SECRET_ACCESS_KEY": "${localEnv:AWS_SECRET_ACCESS_KEY}",
    "AWS_DEFAULT_REGION": "${localEnv:AWS_DEFAULT_REGION}",
    "GITHUB_TOKEN": "${localEnv:GITHUB_TOKEN}"
  }
}
```

### 3. GitHub Actions

**Credential Source**: GitHub Secrets + Action environment

```yaml
# .github/workflows/test.yml
name: Test with DevContainer
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    
    - name: Setup credentials
      env:
        AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
        AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        DOCKER_REGISTRY_TOKEN: ${{ secrets.DOCKER_REGISTRY_TOKEN }}
      run: |
        # Export credentials for devcontainer
        echo "AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}" >> $GITHUB_ENV
        echo "AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}" >> $GITHUB_ENV
        echo "AWS_DEFAULT_REGION=us-west-2" >> $GITHUB_ENV
        echo "GITHUB_TOKEN=${GITHUB_TOKEN}" >> $GITHUB_ENV
        
        # Create Docker config if needed
        if [ -n "${DOCKER_REGISTRY_TOKEN}" ]; then
          docker login your-registry.com -u token -p "${DOCKER_REGISTRY_TOKEN}"
          DOCKER_CONFIG_JSON=$(cat ~/.docker/config.json | base64 -w 0)
          echo "DOCKER_CONFIG_JSON=${DOCKER_CONFIG_JSON}" >> $GITHUB_ENV
        fi
    
    - name: Start DevContainer
      run: |
        devcontainer up --workspace-folder .
        
    - name: Run tests
      run: |
        devcontainer exec --workspace-folder . ./obt test
```

## Common DevContainer Configuration

Create a single `devcontainer.json` that works across all environments:

```json
{
  "name": "OBT Development Environment",
  "dockerComposeFile": "docker-compose.yml",
  "service": "obt",
  "workspaceFolder": "/workspaces/obt",
  
  "containerEnv": {
    "AWS_ACCESS_KEY_ID": "${localEnv:AWS_ACCESS_KEY_ID}",
    "AWS_SECRET_ACCESS_KEY": "${localEnv:AWS_SECRET_ACCESS_KEY}",
    "AWS_SESSION_TOKEN": "${localEnv:AWS_SESSION_TOKEN}",
    "AWS_DEFAULT_REGION": "${localEnv:AWS_DEFAULT_REGION:-us-west-2}",
    "GITHUB_TOKEN": "${localEnv:GITHUB_TOKEN}",
    "DOCKER_CONFIG_JSON": "${localEnv:DOCKER_CONFIG_JSON}",
    "ECR_REGISTRY_URL": "${localEnv:ECR_REGISTRY_URL}"
  },
  
  "postCreateCommand": [
    ".devcontainer/setup-credentials.sh"
  ],
  
  "postStartCommand": [
    ".devcontainer/validate-credentials.sh"  
  ],
  
  "customizations": {
    "vscode": {
      "settings": {
        "terminal.integrated.defaultProfile.linux": "bash"
      }
    }
  }
}
```

## Credential Setup Scripts

### `.devcontainer/setup-credentials.sh`

```bash
#!/bin/bash
set -e

echo "🔐 Setting up credentials..."

# Create credential directories
mkdir -p ~/.aws ~/.config/gh ~/.docker

# Setup AWS credentials
if [ -n "${AWS_ACCESS_KEY_ID}" ] && [ -n "${AWS_SECRET_ACCESS_KEY}" ]; then
    echo "✅ Configuring AWS credentials"
    cat > ~/.aws/credentials << EOF
[default]
aws_access_key_id = ${AWS_ACCESS_KEY_ID}
aws_secret_access_key = ${AWS_SECRET_ACCESS_KEY}
EOF
    
    if [ -n "${AWS_SESSION_TOKEN}" ]; then
        echo "aws_session_token = ${AWS_SESSION_TOKEN}" >> ~/.aws/credentials
    fi
    
    cat > ~/.aws/config << EOF
[default]
region = ${AWS_DEFAULT_REGION:-us-west-2}
output = json
EOF

    # Set permissions
    chmod 600 ~/.aws/credentials ~/.aws/config
else
    echo "⚠️  AWS credentials not provided via environment variables"
    echo "   Checking for instance profile or DevPod credential injection..."
fi

# Setup GitHub token
if [ -n "${GITHUB_TOKEN}" ]; then
    echo "✅ Configuring GitHub CLI"
    echo "${GITHUB_TOKEN}" | gh auth login --with-token
else
    echo "⚠️  GITHUB_TOKEN not provided"
fi

# Setup Docker config
if [ -n "${DOCKER_CONFIG_JSON}" ]; then
    echo "✅ Configuring Docker registry credentials"
    echo "${DOCKER_CONFIG_JSON}" | base64 -d > ~/.docker/config.json
    chmod 600 ~/.docker/config.json
else
    echo "ℹ️  Docker config not provided, relying on DevPod credential injection"
fi

# ECR login if registry URL provided
if [ -n "${ECR_REGISTRY_URL}" ] && command -v aws >/dev/null; then
    echo "✅ Logging into ECR registry: ${ECR_REGISTRY_URL}"
    aws ecr get-login-password --region "${AWS_DEFAULT_REGION:-us-west-2}" | \
        docker login --username AWS --password-stdin "${ECR_REGISTRY_URL}"
fi

echo "🎉 Credential setup complete!"
```

### `.devcontainer/validate-credentials.sh`

```bash
#!/bin/bash
set -e

echo "🔍 Validating credentials..."

# Test AWS credentials
if command -v aws >/dev/null; then
    if aws sts get-caller-identity >/dev/null 2>&1; then
        echo "✅ AWS credentials working"
        aws sts get-caller-identity --output table
    else
        echo "❌ AWS credentials not working"
        exit 1
    fi
fi

# Test GitHub credentials  
if command -v gh >/dev/null; then
    if gh auth status >/dev/null 2>&1; then
        echo "✅ GitHub credentials working"
        gh auth status
    else
        echo "❌ GitHub credentials not working"
    fi
fi

# Test Docker credentials
if docker info >/dev/null 2>&1; then
    echo "✅ Docker daemon accessible"
    if [ -n "${ECR_REGISTRY_URL}" ]; then
        if docker pull "${ECR_REGISTRY_URL}/test:latest" >/dev/null 2>&1 || true; then
            echo "✅ ECR registry accessible"
        fi
    fi
fi

echo "✅ Credential validation complete"
```

## Migration Strategy for OBT

### Phase 1: Add Environment Variable Support

1. **Update OBT to read from environment variables first, fall back to files:**

```bash
# In your OBT scripts, use this pattern:
get_aws_credentials() {
    if [ -n "${AWS_ACCESS_KEY_ID}" ] && [ -n "${AWS_SECRET_ACCESS_KEY}" ]; then
        # Use environment variables
        export AWS_ACCESS_KEY_ID="${AWS_ACCESS_KEY_ID}"
        export AWS_SECRET_ACCESS_KEY="${AWS_SECRET_ACCESS_KEY}"
        export AWS_DEFAULT_REGION="${AWS_DEFAULT_REGION:-us-west-2}"
    elif [ -f ~/.aws/credentials ]; then
        # Fall back to existing file-based credentials
        echo "Using file-based AWS credentials"
    else
        echo "No AWS credentials found"
        exit 1
    fi
}

get_github_token() {
    if [ -n "${GITHUB_TOKEN}" ]; then
        export GITHUB_TOKEN="${GITHUB_TOKEN}"
    elif [ -f ~/.config/gh/hosts.yml ]; then
        # Fall back to existing gh CLI config
        echo "Using existing GitHub CLI configuration"
    else
        echo "No GitHub token found"
        exit 1
    fi
}
```

### Phase 2: Update DevContainer Configuration

2. **Replace volume mounts with environment variable configuration:**

```diff
- "mounts": [
-   "source=${localWorkspaceFolder}/.aws,target=/home/vscode/.aws,type=bind",
-   "source=${localWorkspaceFolder}/.config/gh,target=/home/vscode/.config/gh,type=bind"
- ],
+ "containerEnv": {
+   "AWS_ACCESS_KEY_ID": "${localEnv:AWS_ACCESS_KEY_ID}",
+   "AWS_SECRET_ACCESS_KEY": "${localEnv:AWS_SECRET_ACCESS_KEY}",
+   "GITHUB_TOKEN": "${localEnv:GITHUB_TOKEN}"
+ },
+ "postCreateCommand": ".devcontainer/setup-credentials.sh"
```

### Phase 3: Environment-Specific Setup

3. **Configure each environment:**

**For DevPod AWS:**
```bash
# Use instance profiles (recommended)
devpod provider set-options aws \
  --option AWS_INSTANCE_PROFILE_ARN=arn:aws:iam::ACCOUNT:instance-profile/ObtRole

# Enable credential injection
devpod context set-options default \
  -o SSH_INJECT_GIT_CREDENTIALS=true \
  -o SSH_INJECT_DOCKER_CREDENTIALS=true
```

**For DevPod Local:**
```bash
# Set local environment
export AWS_ACCESS_KEY_ID="your-key"
export AWS_SECRET_ACCESS_KEY="your-secret"
export GITHUB_TOKEN="your-token"
```

**For GitHub Actions:**
```yaml
# Set in repository secrets and use in workflow
env:
  AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
  AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
  GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

## Benefits of This Approach

1. **Unified Configuration**: Single devcontainer.json works everywhere
2. **Secure**: No credential files in containers or repositories  
3. **Flexible**: Each environment provides credentials through its native mechanism
4. **Backward Compatible**: Can gradually migrate from volume mounts
5. **DevOps Friendly**: Works seamlessly in CI/CD pipelines
6. **Auditable**: Clear credential flow and validation

## Next Steps

1. Implement credential reading functions in OBT
2. Add the setup and validation scripts
3. Update devcontainer.json configuration
4. Configure DevPod contexts and providers
5. Update GitHub Actions workflows
6. Test across all three environments
7. Remove volume mount dependencies

This strategy gives you a clear, common foundation that works identically across all your target environments while leveraging each platform's native credential management capabilities.