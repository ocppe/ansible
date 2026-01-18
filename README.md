# OpenShift Platform Engineering Demo - Ansible Automation

Ansible playbooks for setting up the OpenShift Platform Engineering demo infrastructure, including AWS resources, GitHub configuration, and Quay.io container registry.

## Prerequisites

- Python 3.9+
- Ansible Core 2.15+
- AWS CLI configured with appropriate credentials
- OpenShift installer (`openshift-install`), `oc`, and `kubectl`
- GitHub CLI (`gh`) installed and authenticated
- Red Hat pull secret at `~/.pull-secret`
- SSH key at `~/.ssh/ocp_ed25519.pub`

## Quick Start

```shell
# 1. Setup Python environment
cd ansible
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
ansible-galaxy collection install -r requirements.yml

# 2. Set environment variables
export SANDBOX_ID=XXXX
export PULL_SECRET=$(cat ~/.pull-secret)
export SSH_KEY=$(cat ~/.ssh/ocp_ed25519.pub)

# 3. Create Route53 subdomain
ansible-playbook playbooks/route53-setup.yml

# 4. Install OpenShift hub cluster
ansible-playbook playbooks/openshift-install.yml \
  -e cluster_name=hub \
  -e "pull_secret='${PULL_SECRET}'" \
  -e "ssh_public_key='${SSH_KEY}'"

# 5. Setup Quay.io (get QUAY_TOKEN first - see below)
export QUAY_TOKEN=your_quay_oauth_token
ansible-playbook playbooks/quay-setup.yml

# 6. Setup AWS Secrets Manager (after GitOps bootstrap)
source .quay-credentials.env
ansible-playbook playbooks/aws-secrets-setup.yml \
  -e quay_username="${QUAY_USERNAME}" \
  -e quay_password="${QUAY_PASSWORD}" \
  -e git_username=YOUR_GITHUB_USERNAME \
  -e git_password=YOUR_GITHUB_PAT

# 7. Setup GitHub webhooks (after pipelines deployed)
ansible-playbook playbooks/github-webhooks-setup.yml
```

## Playbooks

| Playbook | Description |
|----------|-------------|
| `route53-setup.yml` | Creates Route53 subdomain for OpenShift clusters |
| `openshift-install.yml` | Installs OpenShift cluster on AWS |
| `quay-setup.yml` | Creates Quay.io org, robot account, and repositories |
| `github-repos-setup.yml` | Creates GitHub repositories |
| `aws-secrets-setup.yml` | Creates AWS Secrets Manager secrets and IAM resources |
| `github-webhooks-setup.yml` | Configures GitHub webhooks for CI pipelines |

## Detailed Instructions

### 1. Route53 Setup

Creates the subdomain hosted zone and NS delegation for OpenShift clusters.

```shell
ansible-playbook playbooks/route53-setup.yml
```

This will:
1. Look up the existing `sandbox${SANDBOX_ID}.opentlc.com` hosted zone
2. Create a new hosted zone for `ocp.sandbox${SANDBOX_ID}.opentlc.com`
3. Create NS delegation records in the parent zone

### 2. OpenShift Installation

Installs an OpenShift cluster on AWS.

```shell
ansible-playbook playbooks/openshift-install.yml \
  -e cluster_name=hub \
  -e "pull_secret='${PULL_SECRET}'" \
  -e "ssh_public_key='${SSH_KEY}'"
```

Available clusters: `hub`, `dev`, `test`, `prod`

After installation, cluster artifacts are stored in:
```
.clusters/<cluster_name>/
├── auth/
│   ├── kubeconfig
│   └── kubeadmin-password
└── metadata.json
```

### 3. Quay.io Setup

Creates organization, robot account, and repositories on Quay.io.

**Create Quay.io API Token:**
1. Log in to [quay.io](https://quay.io)
2. Go to **Account Settings** → **Applications**
3. Click **Create New Application**
4. Name it (e.g., `ocppe-setup`)
5. Click on the application, then **Generate Token**
6. Select permissions:
   - Administer Organization
   - Create Repositories
   - Administer Repositories
7. Copy the generated token

```shell
export QUAY_TOKEN=your_oauth_token
ansible-playbook playbooks/quay-setup.yml -e quay_org=ocppe
```

This will:
1. Create the `ocppe` organization (if it doesn't exist)
2. Create a `cicd` robot account
3. Create repositories: `customer-api`, `product-api`, `order-api`
4. Grant the robot account write access to repositories
5. Save credentials to `.quay-credentials.env`

### 4. GitHub Repositories Setup (Optional)

Creates GitHub repositories for the project.

```shell
gh auth login  # If not already authenticated
ansible-playbook playbooks/github-repos-setup.yml -e github_org=ocppe
```

### 5. Bootstrap GitOps

After the hub cluster is installed:

```shell
export KUBECONFIG=$(pwd)/.clusters/hub/auth/kubeconfig

cd ../gitops-hub
oc apply -k bootstrap/

# Wait for operator
oc wait --for=jsonpath='{.status.phase}'=Succeeded csv \
  -n openshift-gitops-operator \
  -l operators.coreos.com/openshift-gitops-operator.openshift-gitops-operator \
  --timeout=300s

# Grant cluster-admin
oc adm policy add-cluster-role-to-user cluster-admin \
  system:serviceaccount:openshift-gitops:openshift-gitops-argocd-application-controller

# Update repoURL in argocd-apps/*.yaml, then deploy
oc apply -f argocd-apps/root-application.yaml
```

### 6. AWS Secrets Manager Setup

Creates secrets and IAM resources for External Secrets Operator.

**Prerequisites:**
- OpenShift cluster running with External Secrets Operator deployed
- Quay credentials (from step 3)
- GitHub Personal Access Token

**Create GitHub PAT:**
1. Go to [GitHub Settings → Developer settings → Personal access tokens](https://github.com/settings/tokens)
2. Click **Generate new token (classic)**
3. Select scopes:
   - `repo` (full control of private repositories)
   - `read:org` (if using organization repos)
4. Copy the token

```shell
cd ../ansible
source .quay-credentials.env

ansible-playbook playbooks/aws-secrets-setup.yml \
  -e quay_username="${QUAY_USERNAME}" \
  -e quay_password="${QUAY_PASSWORD}" \
  -e git_username=YOUR_GITHUB_USERNAME \
  -e git_password=YOUR_GITHUB_PAT
```

This will:
1. Create `ocppe/developer-hub` secret in AWS Secrets Manager
2. Create `ocppe/ci-pipelines` secret in AWS Secrets Manager
3. Create IAM policy `ExternalSecretsPolicy`
4. Create IAM role `ExternalSecretsRole` with OIDC trust
5. Annotate the External Secrets service account
6. Save generated secrets to `.secrets.env`

**Important:** After running, update the Developer Hub OAuthClient with the OAuth secret:

```shell
source .secrets.env
# Update gitops-hub/operators/developer-hub-instance/kustomization.yaml
# Replace REPLACE_WITH_OAUTH_SECRET_FROM_AWS with $OAUTH_CLIENT_SECRET
```

### 7. GitHub Webhooks Setup

Configures webhooks on GitHub repositories for CI pipelines.

**Prerequisites:**
- CI Pipelines deployed on OpenShift (webhook route exists)
- AWS secrets created (step 6)
- GitHub CLI authenticated

```shell
ansible-playbook playbooks/github-webhooks-setup.yml \
  -e github_org=ocppe \
  -e "repos=['customer-api','product-api','order-api']"
```

This will:
1. Get the webhook URL from the OpenShift route
2. Get the webhook secret from AWS Secrets Manager
3. Configure webhooks on each repository with:
   - Content-Type: `application/json`
   - Events: `push`, `pull_request`
   - SSL verification enabled

## Environment Variables Reference

| Variable | Description | Used By |
|----------|-------------|---------|
| `SANDBOX_ID` | OPENTLC sandbox number | All playbooks |
| `PULL_SECRET` | Red Hat pull secret JSON | openshift-install |
| `SSH_KEY` | SSH public key | openshift-install |
| `QUAY_TOKEN` | Quay.io OAuth token | quay-setup |
| `KUBECONFIG` | Path to kubeconfig | aws-secrets-setup, github-webhooks-setup |

## AWS Secrets Structure

| Secret Path | Keys | Purpose |
|-------------|------|---------|
| `ocppe/developer-hub` | `backend-secret`, `oauth-client-secret` | Developer Hub OIDC |
| `ocppe/ci-pipelines` | `github-webhook-secret`, `quay-username`, `quay-password`, `git-username`, `git-password` | CI Pipelines |

## Cluster Configuration

Cluster sizes are configured in `group_vars/all.yml`:

```yaml
clusters:
  hub:
    sizing: sno              # 'sno' or 'ha'
    control_plane:
      count: 1               # 1 for SNO, 3 for HA
      instance_type: c7i.24xlarge
    worker:
      count: 0               # 0 for SNO, 3+ for HA
```

## Domain Structure

| Cluster | Domain |
|---------|--------|
| Hub | hub.ocp.sandbox${SANDBOX_ID}.opentlc.com |
| Dev | dev.ocp.sandbox${SANDBOX_ID}.opentlc.com |
| Test | test.ocp.sandbox${SANDBOX_ID}.opentlc.com |
| Prod | prod.ocp.sandbox${SANDBOX_ID}.opentlc.com |

## Directory Structure

```
ansible/
├── ansible.cfg
├── requirements.txt
├── requirements.yml
├── group_vars/
│   └── all.yml                    # Cluster configurations
├── inventory/
│   └── localhost.yml
├── playbooks/
│   ├── route53-setup.yml          # Route53 subdomain
│   ├── openshift-install.yml      # OpenShift installation
│   ├── quay-setup.yml             # Quay.io setup
│   ├── github-repos-setup.yml     # GitHub repositories
│   ├── aws-secrets-setup.yml      # AWS Secrets Manager + IAM
│   └── github-webhooks-setup.yml  # GitHub webhooks
├── templates/
│   └── install-config.yaml.j2
├── .secrets.env                   # Generated secrets (git-ignored)
└── .quay-credentials.env          # Quay credentials (git-ignored)
```

## Troubleshooting

### AWS Secrets Not Syncing

```shell
# Check ClusterSecretStore
oc describe clustersecretstore aws-secrets-manager

# Check ExternalSecrets
oc get externalsecrets -A

# Verify IAM role annotation
oc get sa external-secrets-sa -n external-secrets-operator -o yaml | grep eks.amazonaws.com
```

### GitHub Webhook Errors

**"Invalid character 'p'"** - Content-Type is wrong. Re-run the webhook setup:
```shell
ansible-playbook playbooks/github-webhooks-setup.yml
```

**403 Forbidden** - Webhook secret mismatch. Verify:
```shell
oc get secret github-webhook-secret -n ci-pipelines -o jsonpath='{.data.webhook-secret}' | base64 -d
```

### Quay.io Push Failures

```shell
# Verify robot credentials in AWS
aws secretsmanager get-secret-value --secret-id ocppe/ci-pipelines --query SecretString --output text | jq .

# Check ExternalSecret
oc describe externalsecret quay-credentials -n ci-pipelines
```

## Platform Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                         Hub Cluster                              │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌────────────┐ │
│  │ ACM         │ │ ACS         │ │ Pipelines   │ │ Dev Hub    │ │
│  │ (Multi-     │ │ (Security)  │ │ (CI/CD)     │ │ (Portal)   │ │
│  │  Cluster)   │ │             │ │             │ │            │ │
│  └─────────────┘ └─────────────┘ └─────────────┘ └────────────┘ │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                    CI Pipelines                             │ │
│  │  GitHub Webhook → Build → Test → Push to quay.io/ocppe     │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  Secrets synced from AWS Secrets Manager via External Secrets   │
└─────────────────────────────────────────────────────────────────┘
         │ ACM manages
         ▼
┌────────────────┐  ┌────────────────┐  ┌────────────────┐
│  Dev Cluster   │  │  Test Cluster  │  │  Prod Cluster  │
└────────────────┘  └────────────────┘  └────────────────┘
```
