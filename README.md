# OpenShift Platform Engineering Demo - Ansible Automation

Ansible playbooks for setting up the OpenShift Platform Engineering demo infrastructure.

## Prerequisites

- Python 3.9+
- Ansible Core 2.15+
- AWS CLI configured with appropriate credentials
- boto3 and botocore Python packages
- OpenShift installer (`openshift-install`), `oc`, and `kubectl` available in PATH
- Red Hat pull secret at `~/.pull-secret` (download from https://console.redhat.com/openshift/install/pull-secret)
- SSH key at `~/.ssh/ocp_ed25519.pub`

## Installation

Create and activate the virtual environment:

```shell
cd ansible
python3 -m venv .venv
source .venv/bin/activate
```

Install Python dependencies:

```shell
pip install -r requirements.txt
```

Install required Ansible collections:

```shell
ansible-galaxy collection install -r requirements.yml
```

## AWS Authentication

Configure AWS credentials:

```shell
aws configure
```

## Playbooks

### Environment Variables

Set the required environment variables before running any playbooks:

```shell
export SANDBOX_ID=XXXX
export PULL_SECRET=$(cat ~/.pull-secret)
export SSH_KEY=$(cat ~/.ssh/ocp_ed25519.pub)
```

### 1. Route53 Setup

Creates the subdomain hosted zone and NS delegation for the OpenShift clusters.

```shell
ansible-playbook playbooks/route53-setup.yml
```

This will:

1. Look up the existing `sandbox${SANDBOX_ID}.opentlc.com` hosted zone
2. Create a new hosted zone for `ocp.sandbox${SANDBOX_ID}.opentlc.com`
3. Create NS delegation records in the parent zone

### 2. OpenShift Install

Installs an OpenShift cluster on AWS.

```shell
# Install a cluster (hub, dev, test, prod)
ansible-playbook playbooks/openshift-install.yml \
  -e cluster_name=hub \
  -e "pull_secret='${PULL_SECRET}'" \
  -e "ssh_public_key='${SSH_KEY}'"
```

## Cluster Configuration

Cluster sizes are configured in `group_vars/all.yml`. Each cluster can be configured as:

- **SNO (Single Node OpenShift)**: 1 control plane node, 0 workers - default for demo
- **HA (High Availability)**: 3 control plane nodes, 3 workers

To switch a cluster to HA, edit `group_vars/all.yml`:

```yaml
clusters:
  hub:
    sizing: ha              # Change from 'sno' to 'ha'
    control_plane:
      count: 3              # Change from 1 to 3
      instance_type: m6i.4xlarge
    worker:
      count: 3              # Change from 0 to 3
      instance_type: m6i.2xlarge
```

## Domain Structure

| Cluster | Domain                                    |
| ------- | ----------------------------------------- |
| Hub     | hub.ocp.sandbox${SANDBOX_ID}.opentlc.com  |
| Dev     | dev.ocp.sandbox${SANDBOX_ID}.opentlc.com  |
| Test    | test.ocp.sandbox${SANDBOX_ID}.opentlc.com |
| Prod    | prod.ocp.sandbox${SANDBOX_ID}.opentlc.com |

## Directory Structure

```
ansible/
├── ansible.cfg
├── requirements.yml
├── group_vars/
│   └── all.yml              # Cluster configurations
├── inventory/
│   └── localhost.yml
├── playbooks/
│   ├── route53-setup.yml     # Route53 subdomain setup
│   └── openshift-install.yml # OpenShift cluster installation
└── templates/
    └── install-config.yaml.j2
```

After installation, cluster artifacts are stored in:

```
.clusters/
└── <cluster_name>/
    ├── auth/
    │   ├── kubeconfig
    │   └── kubeadmin-password
    ├── metadata.json
    └── install-config.yaml.backup
```

## Post-Installation

After the hub cluster is installed, bootstrap the GitOps configuration:

```shell
# Set kubeconfig
export KUBECONFIG=$(pwd)/.clusters/hub/auth/kubeconfig

# Bootstrap OpenShift GitOps (see gitops-hub repository)
cd ../gitops-hub
oc apply -k bootstrap/
```

### Create AWS Secrets Manager Secrets

All platform secrets are managed via External Secrets Operator syncing from AWS Secrets Manager. Create these secrets before deploying the app-of-apps:

```shell
# Developer Hub secrets
BACKEND_SECRET=$(openssl rand -base64 32)
OAUTH_SECRET=$(openssl rand -base64 32)
aws secretsmanager create-secret \
  --name ocppe/developer-hub \
  --secret-string "{
    \"backend-secret\": \"${BACKEND_SECRET}\",
    \"oauth-client-secret\": \"${OAUTH_SECRET}\"
  }"

# CI Pipeline secrets
WEBHOOK_SECRET=$(openssl rand -hex 20)
aws secretsmanager create-secret \
  --name ocppe/ci-pipelines \
  --secret-string "{
    \"github-webhook-secret\": \"${WEBHOOK_SECRET}\",
    \"quay-username\": \"YOUR_QUAY_USERNAME\",
    \"quay-password\": \"YOUR_QUAY_ROBOT_TOKEN\",
    \"git-username\": \"YOUR_GITHUB_USERNAME\",
    \"git-password\": \"YOUR_GITHUB_PAT\"
  }"

# Save the webhook secret for GitHub webhook configuration
echo "GitHub Webhook Secret: ${WEBHOOK_SECRET}"
```

### Deploy the App-of-Apps

```shell
# Update repoURL in argocd-apps/*.yaml to your Git repository
# Then deploy the root application
oc apply -f argocd-apps/root-application.yaml
```

See the `gitops-hub` repository README for complete instructions on deploying:

- External Secrets Operator (syncs secrets from AWS Secrets Manager)
- Advanced Cluster Management
- Advanced Cluster Security
- OpenShift Pipelines
- Red Hat Developer Hub
- CI Pipelines

## Platform Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                         Hub Cluster                              │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌────────────┐ │
│  │ ACM         │ │ ACS         │ │ Pipelines   │ │ Dev Hub    │ │
│  │ (Multi-     │ │ (Security)  │ │ (CI/CD)     │ │ (Portal)   │ │
│  │  Cluster)   │ │             │ │             │ │            │ │
│  └─────────────┘ └─────────────┘ └─────────────┘ └────────────┘ │
│                            │                                     │
│                    GitHub Webhooks                               │
│                            │                                     │
│  ┌─────────────────────────┴─────────────────────────────────┐  │
│  │                    CI Pipelines                            │  │
│  │  customer-api → build → test → quay.io/ocppe/customer-api │  │
│  │  product-api  → build → test → quay.io/ocppe/product-api  │  │
│  │  order-api    → build → test → quay.io/ocppe/order-api    │  │
│  └────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
         │ ACM manages
         ▼
┌────────────────┐  ┌────────────────┐  ┌────────────────┐
│  Dev Cluster   │  │  Test Cluster  │  │  Prod Cluster  │
│                │  │                │  │                │
│ - customer-api │  │ - customer-api │  │ - customer-api │
│ - product-api  │  │ - product-api  │  │ - product-api  │
│ - order-api    │  │ - order-api    │  │ - order-api    │
└────────────────┘  └────────────────┘  └────────────────┘
```
