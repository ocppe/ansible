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
clusters/
└── <cluster_name>/
    ├── auth/
    │   ├── kubeconfig
    │   └── kubeadmin-password
    ├── metadata.json
    └── install-config.yaml.backup
```
