# Corrected Role-Based Strategy: Handling Missing Mounts

## The Problem

The devcontainer.json mount configuration I previously suggested will fail when paths don't exist:

```json
"mounts": [
  {
    "source": "${env:HOME}/.aws/sso",    # ❌ Fails if path doesn't exist
    "target": "/home/vscode/.aws/sso",
    "type": "bind"
  }
]
```

**When mounts fail:**
- DevPod AWS: Local `~/.aws/sso` doesn't exist on remote EC2
- GitHub Actions: Runner doesn't have SSO cache
- Fresh setups: New developers haven't run `aws sso login`
- Different OS: Windows vs Linux path differences

## Solution 1: Environment-Specific DevContainer Files (Recommended)

Create multiple devcontainer configurations and let each environment choose:

### Directory Structure
```
.devcontainer/
├── devcontainer.json                    # Default (local development)
├── devcontainer-aws.json               # DevPod AWS (no mounts)
├── devcontainer-actions.json           # GitHub Actions (no mounts)
└── setup-credentials.sh                # Common setup script
```

### `.devcontainer/devcontainer.json` (Local Development)
```json
{
  "name": "OBT Local Development",
  "dockerComposeFile": "../docker-compose.yml",
  "service": "obt",
  "workspaceFolder": "/workspaces/obt",
  
  "mounts": [
    {
      "source": "${localWorkspaceFolder}/.aws/config",
      "target": "/home/vscode/.aws/config",
      "type": "bind"
    },
    {
      "source": "${env:HOME}/.aws/sso",
      "target": "/home/vscode/.aws/sso", 
      "type": "bind"
    }
  ],
  
  "containerEnv": {
    "AWS_DEFAULT_REGION": "${localEnv:AWS_DEFAULT_REGION:-us-west-2}",
    "AWS_PROFILE": "${localEnv:AWS_PROFILE}",
    "GITHUB_TOKEN": "${localEnv:GITHUB_TOKEN}",
    "OBT_ENVIRONMENT": "local"
  },
  
  "postCreateCommand": ".devcontainer/setup-credentials.sh"
}
```

### `.devcontainer/devcontainer-aws.json` (DevPod AWS - No Mounts)
```json
{
  "name": "OBT AWS DevPod",
  "dockerComposeFile": "../docker-compose.yml", 
  "service": "obt",
  "workspaceFolder": "/workspaces/obt",
  
  "containerEnv": {
    "AWS_DEFAULT_REGION": "us-west-2",
    "GITHUB_TOKEN": "${localEnv:GITHUB_TOKEN}",
    "OBT_ENVIRONMENT": "aws"
  },
  
  "postCreateCommand": ".devcontainer/setup-credentials.sh"
}
```

### `.devcontainer/devcontainer-actions.json` (GitHub Actions - No Mounts)
```json
{
  "name": "OBT GitHub Actions",
  "dockerComposeFile": "../docker-compose.yml",
  "service": "obt", 
  "workspaceFolder": "/workspaces/obt",
  
  "containerEnv": {
    "AWS_DEFAULT_REGION": "us-west-2",
    "GITHUB_TOKEN": "${localEnv:GITHUB_TOKEN}",
    "OBT_ENVIRONMENT": "actions"
  },
  
  "postCreateCommand": ".devcontainer/setup-credentials.sh"
}
```

### Usage by Environment

**Local DevPod:**
```bash
devpod up .  # Uses default .devcontainer/devcontainer.json
```

**DevPod AWS:**
```bash
devpod up . --devcontainer-path .devcontainer/devcontainer-aws.json
```

**GitHub Actions:**
```yaml
- name: Start DevContainer
  run: |
    devcontainer up --workspace-folder . \
      --config .devcontainer/devcontainer-actions.json
```

## Solution 2: Conditional Mounts with Try/Catch Setup

Create a single devcontainer.json that attempts mounts but handles failures gracefully:

### Single `devcontainer.json` with Optional Mounts
```json
{
  "name": "OBT Development Environment",
  "dockerComposeFile": "docker-compose.yml",
  "service": "obt",
  "workspaceFolder": "/workspaces/obt",
  
  "containerEnv": {
    "AWS_DEFAULT_REGION": "${localEnv:AWS_DEFAULT_REGION:-us-west-2}",
    "AWS_PROFILE": "${localEnv:AWS_PROFILE}",
    "GITHUB_TOKEN": "${localEnv:GITHUB_TOKEN}",
    
    "AWS_CONFIG_MOUNT": "${localWorkspaceFolder}/.aws/config",
    "AWS_SSO_MOUNT": "${env:HOME}/.aws/sso"
  },
  
  "initializeCommand": ".devcontainer/prepare-mounts.sh",
  "postCreateCommand": ".devcontainer/setup-credentials.sh"
}
```

### `.devcontainer/prepare-mounts.sh`
```bash
#!/bin/bash
set -e

echo "🔧 Preparing mount directories..."

# Ensure local mount sources exist or create empty ones
if [[ -n "${AWS_CONFIG_MOUNT:-}" ]] && [[ ! -f "${AWS_CONFIG_MOUNT}" ]]; then
    echo "Creating empty AWS config for mount: ${AWS_CONFIG_MOUNT}"
    mkdir -p "$(dirname "${AWS_CONFIG_MOUNT}")"
    touch "${AWS_CONFIG_MOUNT}"
fi

if [[ -n "${AWS_SSO_MOUNT:-}" ]] && [[ ! -d "${AWS_SSO_MOUNT}" ]]; then
    echo "Creating empty SSO directory for mount: ${AWS_SSO_MOUNT}"
    mkdir -p "${AWS_SSO_MOUNT}"
fi

echo "✅ Mount preparation complete"
```

**Problem with Solution 2**: Still doesn't solve the core issue that DevPod AWS evaluates local paths that don't exist on remote EC2.

## Solution 3: Runtime Detection and Setup (Cleanest)

Use a single devcontainer.json with **no problematic mounts** and handle everything in setup:

### Single `devcontainer.json` (No Mounts)
```json
{
  "name": "OBT Development Environment", 
  "dockerComposeFile": "docker-compose.yml",
  "service": "obt",
  "workspaceFolder": "/workspaces/obt",
  
  "containerEnv": {
    "AWS_DEFAULT_REGION": "${localEnv:AWS_DEFAULT_REGION:-us-west-2}",
    "AWS_PROFILE": "${localEnv:AWS_PROFILE}",  
    "GITHUB_TOKEN": "${localEnv:GITHUB_TOKEN}"
  },
  
  "postCreateCommand": ".devcontainer/setup-credentials.sh"
}
```

### `.devcontainer/setup-credentials.sh` (Handles All Scenarios)
```bash
#!/bin/bash
set -e

echo "🔐 Setting up role-based credentials..."

# Detect environment
if [[ -n "${AWS_CONTAINER_CREDENTIALS_RELATIVE_URI}" ]] || [[ -n "${AWS_CONTAINER_CREDENTIALS_FULL_URI}" ]]; then
    ENVIRONMENT="ec2-instance-profile"
elif [[ "${GITHUB_ACTIONS}" == "true" ]]; then
    ENVIRONMENT="github-actions"  
elif [[ "${USER}" == "vscode" ]] && [[ -n "${AWS_PROFILE}" ]]; then
    ENVIRONMENT="devpod-local"
else
    ENVIRONMENT="local-development"
fi

echo "📍 Detected environment: ${ENVIRONMENT}"

# Create necessary directories
mkdir -p ~/.aws ~/.config/gh ~/.docker

case "${ENVIRONMENT}" in
    "ec2-instance-profile")
        echo "✅ Using EC2 instance profile for AWS authentication"
        # Instance profile credentials are automatically available
        
        # Set default region
        cat > ~/.aws/config << EOF
[default]
region = ${AWS_DEFAULT_REGION:-us-west-2}
output = json
EOF
        ;;
        
    "github-actions")
        echo "✅ Using GitHub Actions OIDC for AWS authentication" 
        # AWS credentials configured by aws-actions/configure-aws-credentials
        
        # Ensure config exists
        cat > ~/.aws/config << EOF
[default]
region = ${AWS_DEFAULT_REGION:-us-west-2}
output = json
EOF
        ;;
        
    "devpod-local"|"local-development")
        echo "✅ Setting up for local AWS SSO authentication"
        
        # Try to copy AWS config from host if mounted
        if [[ -f /host-aws-config ]] 2>/dev/null; then
            echo "Copying AWS config from host mount"
            cp /host-aws-config ~/.aws/config
        fi
        
        # Try to copy SSO cache from host if mounted  
        if [[ -d /host-aws-sso ]] 2>/dev/null; then
            echo "Copying AWS SSO cache from host mount"
            cp -r /host-aws-sso/* ~/.aws/sso/ 2>/dev/null || true
        fi
        
        # If no mounted config, create basic one
        if [[ ! -f ~/.aws/config ]]; then
            echo "Creating basic AWS config"
            cat > ~/.aws/config << EOF
[default]
region = ${AWS_DEFAULT_REGION:-us-west-2}
output = json

# Add your SSO profiles here, or mount your existing config
EOF
        fi
        
        # Validate SSO session if profile is set
        if [[ -n "${AWS_PROFILE}" ]]; then
            echo "🔍 Checking SSO session for profile: ${AWS_PROFILE}"
            if ! aws sts get-caller-identity --profile "${AWS_PROFILE}" >/dev/null 2>&1; then
                echo "⚠️  SSO session expired or invalid for profile: ${AWS_PROFILE}" 
                echo "   Run outside container: aws sso login --profile ${AWS_PROFILE}"
                echo "   Then restart the devcontainer"
            fi
        fi
        ;;
esac

# Setup GitHub CLI
if [[ -n "${GITHUB_TOKEN}" ]]; then
    echo "✅ Configuring GitHub CLI"
    echo "${GITHUB_TOKEN}" | gh auth login --with-token
elif [[ "${ENVIRONMENT}" == "github-actions" ]]; then
    echo "✅ GitHub CLI will use Actions token automatically"
else
    echo "⚠️  GITHUB_TOKEN not provided"
fi

echo "🎉 Credential setup complete!"
```

## Solution 4: Docker Compose Override (Most Flexible)

Use Docker Compose overrides for environment-specific configurations:

### `docker-compose.yml` (Base)
```yaml
version: '3.8'
services:
  obt:
    image: your-obt-image
    working_dir: /workspaces/obt
    command: sleep infinity
    environment:
      - AWS_DEFAULT_REGION=${AWS_DEFAULT_REGION:-us-west-2}
      - AWS_PROFILE=${AWS_PROFILE}
      - GITHUB_TOKEN=${GITHUB_TOKEN}
```

### `docker-compose.local.yml` (Local Override)
```yaml
version: '3.8' 
services:
  obt:
    volumes:
      - ${HOME}/.aws/config:/home/vscode/.aws/config:ro
      - ${HOME}/.aws/sso:/home/vscode/.aws/sso:ro
```

### `docker-compose.aws.yml` (AWS Override - No Volumes)
```yaml
version: '3.8'
services:
  obt:
    environment:
      - OBT_ENVIRONMENT=aws
```

### Usage

**Local DevPod:**
```json
{
  "dockerComposeFile": ["docker-compose.yml", "docker-compose.local.yml"]
}
```

**DevPod AWS:**
```json  
{
  "dockerComposeFile": ["docker-compose.yml", "docker-compose.aws.yml"]
}
```

## Recommendation: Solution 1 (Environment-Specific DevContainers)

**Best approach for your use case:**

1. **Create 3 devcontainer files** for each environment
2. **Use explicit configuration** rather than trying to be too clever  
3. **DevPod AWS specifies the AWS config**: `devpod up . --devcontainer-path .devcontainer/devcontainer-aws.json`
4. **Clear separation of concerns** - each environment gets exactly what it needs

This approach:
- ✅ **Eliminates mount failures** - only mount what exists
- ✅ **Clear and explicit** - no guessing about environment  
- ✅ **Easy to debug** - each environment has its own config
- ✅ **Flexible** - can customize each environment independently

The key insight is that **trying to use one devcontainer.json for all environments** introduces too much complexity. Better to be explicit about what each environment needs.