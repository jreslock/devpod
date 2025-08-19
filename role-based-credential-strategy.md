# Role-Based Unified Credential Strategy for OBT

## Overview

This strategy leverages AWS roles everywhere, eliminating the need for long-term access keys:

- **Local Development**: AWS SSO profiles that assume roles
- **DevPod + AWS Provider**: IAM instance profiles (roles attached to EC2)
- **GitHub Actions**: OIDC authentication to assume roles

The common element is that AWS CLI always uses roles, but the assumption mechanism varies by environment.

## Core Principle: Role-Based Authentication Contract

Instead of managing access keys, define a standard set of role-based environment variables:

```bash
# AWS Role Configuration
AWS_DEFAULT_REGION=us-west-2
AWS_PROFILE=your-sso-profile          # Local/DevPod local only
AWS_ROLE_ARN=arn:aws:iam::ACCOUNT:role/ObtRole  # GitHub Actions only

# GitHub Authentication  
GITHUB_TOKEN=ghp_token_or_from_actions

# Optional: Explicit role assumption
AWS_ROLE_SESSION_NAME=obt-session     # For GitHub Actions role assumption
```

## Implementation Strategy by Environment

### 1. Local Development (DevPod + Local Docker)

**Credential Source**: Existing AWS SSO + ~/.aws/config

Users continue their normal workflow:
```bash
# Users already do this
aws sso login --profile my-dev-profile
```

**Updated devcontainer.json**:
```json
{
  "name": "OBT Development Environment",
  "dockerComposeFile": "docker-compose.yml",
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
    "GITHUB_TOKEN": "${localEnv:GITHUB_TOKEN}"
  },
  
  "postCreateCommand": ".devcontainer/setup-credentials.sh"
}
```

**Why keep volume mounts for local development?**
- AWS SSO cache lives in `~/.aws/sso/cache/`
- SSO tokens are stored as files, not environment variables
- DevPod's local Docker provider supports volume mounts perfectly
- Users expect their existing `aws sso login` workflow to work

### 2. DevPod + AWS Provider (Remote EC2)

**Credential Source**: IAM Instance Profiles

```bash
# Configure AWS provider with instance profile
devpod provider set-options aws \
  --option AWS_INSTANCE_PROFILE_ARN=arn:aws:iam::ACCOUNT:instance-profile/ObtDevPodProfile

# Enable other credential injection
devpod context set-options default \
  -o SSH_INJECT_GIT_CREDENTIALS=true \
  -o SSH_INJECT_DOCKER_CREDENTIALS=true
```

**IAM Instance Profile Setup**:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage",
        "s3:GetObject",
        "s3:PutObject",
        "s3:ListBucket"
      ],
      "Resource": "*"
    }
  ]
}
```

**devcontainer.json for AWS** (no volume mounts needed):
```json
{
  "name": "OBT Development Environment", 
  "dockerComposeFile": "docker-compose.yml",
  "service": "obt",
  "workspaceFolder": "/workspaces/obt",
  
  "containerEnv": {
    "AWS_DEFAULT_REGION": "us-west-2",
    "GITHUB_TOKEN": "${localEnv:GITHUB_TOKEN}",
    "OBT_ENVIRONMENT": "devpod-aws"
  },
  
  "postCreateCommand": ".devcontainer/setup-credentials.sh"
}
```

### 3. GitHub Actions (OIDC Role Assumption)

**Credential Source**: OIDC authentication to assume AWS role

**GitHub OIDC Identity Provider Setup** (one-time):
```bash
# Create OIDC identity provider in AWS
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com \
  --thumbprint-list 6938fd4d98bab03faadb97b34396831e3780aea1
```

**IAM Role for GitHub Actions**:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::ACCOUNT:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:om1inc/obt:*"
        }
      }
    }
  ]
}
```

**GitHub Actions Workflow**:
```yaml
name: Test with DevContainer
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    permissions:
      id-token: write   # Required for OIDC
      contents: read
      
    steps:
    - uses: actions/checkout@v4
    
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v4
      with:
        role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/GitHubActionsObtRole
        role-session-name: obt-github-actions
        aws-region: us-west-2
        
    - name: Start DevContainer
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        OBT_ENVIRONMENT: github-actions
      run: |
        devcontainer up --workspace-folder .
        
    - name: Run tests
      run: |
        devcontainer exec --workspace-folder . ./obt test
```

## Environment Detection Strategy

Create a smart devcontainer.json that adapts based on environment:

### Single Smart devcontainer.json

```json
{
  "name": "OBT Development Environment",
  "dockerComposeFile": "docker-compose.yml", 
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
    "OBT_ENVIRONMENT": "${localEnv:OBT_ENVIRONMENT:-local}"
  },
  
  "postCreateCommand": ".devcontainer/setup-credentials.sh",
  "postStartCommand": ".devcontainer/validate-credentials.sh"
}
```

**Note**: Volume mounts will be ignored in DevPod AWS (since local paths don't exist), but work perfectly in DevPod local Docker.

## Smart Credential Setup Script

### `.devcontainer/setup-credentials.sh`

```bash
#!/bin/bash
set -e

echo "🔐 Setting up role-based credentials..."

# Detect environment
if [[ -n "${AWS_CONTAINER_CREDENTIALS_RELATIVE_URI}" ]] || [[ -n "${AWS_CONTAINER_CREDENTIALS_FULL_URI}" ]]; then
    ENVIRONMENT="ec2-instance-profile"
elif [[ "${GITHUB_ACTIONS}" == "true" ]]; then
    ENVIRONMENT="github-actions"
elif [[ -f ~/.aws/config ]] && [[ -d ~/.aws/sso ]]; then
    ENVIRONMENT="local-sso"
else
    ENVIRONMENT="unknown"
fi

echo "📍 Detected environment: ${ENVIRONMENT}"

case "${ENVIRONMENT}" in
    "ec2-instance-profile")
        echo "✅ Using EC2 instance profile for AWS authentication"
        # Instance profile credentials are automatically available
        # No setup needed - AWS CLI will use them automatically
        ;;
        
    "github-actions")
        echo "✅ Using GitHub Actions OIDC for AWS authentication"
        # AWS credentials should already be configured by aws-actions/configure-aws-credentials
        ;;
        
    "local-sso")
        echo "✅ Using AWS SSO for authentication"
        
        # Ensure AWS config exists
        if [[ ! -f ~/.aws/config ]]; then
            echo "❌ ~/.aws/config not found"
            echo "Please ensure your AWS SSO config is volume mounted or configured"
            exit 1
        fi
        
        # Check if SSO session is valid
        if [[ -n "${AWS_PROFILE}" ]]; then
            echo "🔍 Checking SSO session for profile: ${AWS_PROFILE}"
            if ! aws sts get-caller-identity --profile "${AWS_PROFILE}" >/dev/null 2>&1; then
                echo "❌ SSO session expired or invalid for profile: ${AWS_PROFILE}"
                echo "Please run: aws sso login --profile ${AWS_PROFILE}"
                echo "Note: You may need to run this outside the container"
                exit 1
            fi
        else
            echo "⚠️  AWS_PROFILE not set, using default profile"
        fi
        ;;
        
    "unknown")
        echo "❌ Unable to detect AWS authentication method"
        echo "Expected one of:"
        echo "  - EC2 instance profile (AWS_CONTAINER_CREDENTIALS_*)"
        echo "  - GitHub Actions (GITHUB_ACTIONS=true)"
        echo "  - Local SSO (~/.aws/config + ~/.aws/sso)"
        exit 1
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

# Create any missing directories
mkdir -p ~/.aws ~/.config/gh ~/.docker

echo "🎉 Role-based credential setup complete!"
```

### `.devcontainer/validate-credentials.sh`

```bash
#!/bin/bash
set -e

echo "🔍 Validating role-based credentials..."

# Test AWS credentials
if command -v aws >/dev/null; then
    echo "🧪 Testing AWS credentials..."
    
    if IDENTITY=$(aws sts get-caller-identity 2>/dev/null); then
        echo "✅ AWS authentication successful"
        echo "${IDENTITY}" | jq -r '"\(.UserId) - \(.Arn)"' || echo "${IDENTITY}"
        
        # Show what type of credentials we're using
        USER_ID=$(echo "${IDENTITY}" | jq -r '.UserId')
        if [[ "${USER_ID}" == *":user/"* ]]; then
            echo "📋 Using IAM user credentials"
        elif [[ "${USER_ID}" == *":assumed-role/"* ]]; then
            echo "📋 Using assumed role credentials (✅ recommended)"
        elif [[ "${USER_ID}" == AIDA* ]]; then
            echo "📋 Using IAM user access keys"
        elif [[ "${USER_ID}" == AKIA* ]]; then
            echo "📋 Using long-term access keys"
        fi
    else
        echo "❌ AWS authentication failed"
        
        if [[ -n "${AWS_PROFILE}" ]]; then
            echo "Trying with profile: ${AWS_PROFILE}"
            aws sts get-caller-identity --profile "${AWS_PROFILE}" || echo "Profile authentication also failed"
        fi
        
        exit 1
    fi
fi

# Test GitHub credentials
if command -v gh >/dev/null; then
    if gh auth status >/dev/null 2>&1; then
        echo "✅ GitHub authentication successful"
        gh auth status
    else
        echo "⚠️  GitHub authentication not configured"
    fi
fi

# Test ECR access if relevant
if command -v docker >/dev/null && [[ -n "${ECR_REGISTRY_URL:-}" ]]; then
    echo "🧪 Testing ECR access..."
    if aws ecr get-login-password --region "${AWS_DEFAULT_REGION}" | \
       docker login --username AWS --password-stdin "${ECR_REGISTRY_URL}" >/dev/null 2>&1; then
        echo "✅ ECR authentication successful"
    else
        echo "⚠️  ECR authentication failed"
    fi
fi

echo "✅ Credential validation complete"
```

## OBT Integration

Update your OBT scripts to work with role-based authentication:

```bash
# In your OBT utility
setup_aws_auth() {
    echo "🔧 Configuring AWS authentication..."
    
    # Check if we can authenticate
    if ! aws sts get-caller-identity >/dev/null 2>&1; then
        if [[ -n "${AWS_PROFILE}" ]]; then
            echo "Trying with profile: ${AWS_PROFILE}"
            export AWS_PROFILE="${AWS_PROFILE}"
            if ! aws sts get-caller-identity --profile "${AWS_PROFILE}" >/dev/null 2>&1; then
                echo "❌ AWS authentication failed with profile ${AWS_PROFILE}"
                echo "Try: aws sso login --profile ${AWS_PROFILE}"
                return 1
            fi
        else
            echo "❌ AWS authentication failed and no profile specified"
            return 1
        fi
    fi
    
    echo "✅ AWS authentication verified"
    aws sts get-caller-identity --output table
}

# Use this at the start of OBT operations
setup_aws_auth || exit 1
```

## Migration Benefits

✅ **Security**: No long-term credentials anywhere  
✅ **Consistency**: Roles used across all environments  
✅ **User Experience**: Local users keep their existing SSO workflow  
✅ **Auditability**: Clear role-based access patterns  
✅ **Scalability**: Easy to manage permissions via role policies  

## Migration Steps

1. **Set up IAM roles and instance profiles for DevPod AWS**
2. **Configure GitHub OIDC and role assumption**  
3. **Update devcontainer.json with smart environment detection**
4. **Add credential setup and validation scripts**
5. **Update OBT to verify AWS authentication on startup**
6. **Test across all three environments**

This approach respects your existing AWS SSO workflow while extending the same role-based security model to remote environments.
