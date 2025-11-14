# Execution Environment for AIOps Demo

TBD, this is not finished and working yet.

This directory contains the configuration files needed to build a custom Execution Environment (EE) for the AIOps demo using `ansible-builder` and `ansible-navigator`.

## Overview

An Execution Environment is a container image that includes:
- Ansible Core
- Ansible Collections
- Python dependencies
- System packages
- Other dependencies needed to run your automation

## Files in this Directory

- `execution-environment.yml` - Main EE definition file
- `requirements.yml` - Ansible Galaxy collections to install
- `ansible.cfg` - Configuration for accessing Red Hat Automation Hub
- `python-requirements.txt` - (Optional) Additional Python packages
- `bindep.txt` - (Optional) Additional system packages

## Prerequisites

### Required Tools

```bash
# Install ansible-builder
pip install ansible-builder

# Install ansible-navigator
pip install ansible-navigator

# Install podman (container runtime)
# On RHEL/Fedora:
sudo dnf install podman

# On Ubuntu/Debian:
sudo apt install podman
```

### Red Hat Registry Authentication

You need access to the Red Hat Container Registry to pull the base image.

#### Using Red Hat Account Credentials

```bash
podman login registry.redhat.io
# Enter your Red Hat account username and password
```

### Red Hat Automation Hub Token

To download certified collections from Red Hat Automation Hub, you need an API token:

1. Go to https://console.redhat.com/ansible/automation-hub/token
2. Click "Load token"
3. Copy your API token

#### Setting Your Token (Choose ONE method)

**Method 1: Environment Variable (Recommended for CI/CD)**
```bash
TOKEN="your-token-here"
export ANSIBLE_GALAXY_SERVER_AUTOMATION_HUB_VALIDATED_TOKEN="${TOKEN}"
export ANSIBLE_GALAXY_SERVER_AUTOMATION_HUB_PUBLISHED_TOKEN="${TOKEN}"
```

## Building the Execution Environment

### Basic Build

```bash
# Change to this directory
cd demo_aiops/execution-environment

# Build the EE
ansible-builder build \
                --build-arg ANSIBLE_GALAXY_SERVER_AUTOMATION_HUB_VALIDATED_TOKEN=${ANSIBLE_GALAXY_SERVER_AUTOMATION_HUB_VALIDATED_TOKEN} \
                --build-arg ANSIBLE_GALAXY_SERVER_AUTOMATION_HUB_PUBLISHED_TOKEN=${ANSIBLE_GALAXY_SERVER_AUTOMATION_HUB_PUBLISHED_TOKEN} \
                -t aiops-ee:latest -v 3
```

The `-v 3` flag enables verbose output, which is helpful for debugging.

## Using the Execution Environment

### With ansible-navigator

Create or update your `ansible-navigator.yml`:

```yaml
---
ansible-navigator:
  execution-environment:
    enabled: true
    image: localhost/aiops-ee:latest
    pull:
      policy: missing

  playbook-artifact:
    enable: true
    save-as: /tmp/playbook-artifacts/{playbook_name}-artifact-{ts_utc}.json
```

Run a playbook:

```bash
ansible-navigator run ../playbooks/post_setup.yml
```

### Testing the EE

Test that your EE works correctly:

```bash
# Run a simple ad-hoc command
ansible-navigator exec -- ansible localhost -m ping

# List installed collections
ansible-navigator exec -- ansible-galaxy collection list

# Check Python packages
ansible-navigator exec -- pip list
```

## Pushing to a Registry

### Push to Quay.io

```bash
# Tag the image
podman tag localhost/aiops-ee:latest quay.io/YOUR_USERNAME/aiops-ee:latest

# Login to Quay
podman login quay.io

# Push the image
podman push quay.io/YOUR_USERNAME/aiops-ee:latest
```

### Push to Private Registry

```bash
# Tag the image
podman tag localhost/aiops-ee:latest registry.example.com/aiops-ee:latest

# Login to your registry
podman login registry.example.com

# Push the image
podman push registry.example.com/aiops-ee:latest
```

## Automation with Ansible

You can automate EE builds using the `infra.ee_utilities` collection:

```yaml
---
- name: Build AIOps Execution Environment
  hosts: localhost
  connection: local
  gather_facts: true
  collections:
    - infra.ee_utilities

  vars:
    ee_builder_dir: "{{ playbook_dir }}/execution-environment"
    ee_registry_dest: "quay.io/your-username"
    ee_registry_username: "your-username"
    ee_registry_password: "{{ lookup('env', 'QUAY_PASSWORD') }}"
    ee_verbosity: 3

    ee_list:
      - name: aiops-ee
        tag: latest
        images:
          base_image:
            name: registry.redhat.io/ansible-automation-platform/ee-minimal-rhel9:2.16.5-2

  tasks:
    - name: Build and push EE
      ansible.builtin.include_role:
        name: infra.ee_utilities.ee_builder
```
