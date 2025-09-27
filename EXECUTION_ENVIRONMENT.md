# Ansible Execution Environment for CISA ED 25-03

This repository includes an Ansible execution environment (EE) that provides a containerized runtime for the CISA ED 25-03 collection with all necessary dependencies pre-installed.

## What is an Execution Environment?

An Ansible execution environment is a container image that includes:
- Ansible Core
- Python dependencies
- System packages
- Ansible collections
- Custom configuration

This ensures consistent execution across different environments and simplifies deployment.

## Image Location

The execution environment is automatically built and published to GitHub Container Registry (GHCR):

```
ghcr.io/kush-gupt/ansible-ed-25-03/ansible-ee:latest
```

## Included Components

### Base Image
- `quay.io/ubi9/ubi:latest` - Red Hat Universal Base Image 9

### Ansible Core
- `ansible-core` (>=2.15.0, <2.18.0) - Installed via pip with version constraints

### Ansible Collections
- `cisco.asa` (>=5.0.0)
- `ansible.netcommon` (>=6.0.0)

### Python Dependencies
- `pyyaml` (>=6.0)
- `jinja2` (>=3.0.0)
- `requests` (>=2.25.0)
- `netaddr` (>=0.8.0)
- `paramiko` (>=2.7.0)
- `cryptography` (>=3.0.0)

### System Packages
- `python3` and `python3-pip`
- `git`
- `openssh-clients`/`openssh-client`
- `gcc` and development headers for compiling Python packages

## Usage Examples

### With Podman

```bash
# Pull the image
podman pull ghcr.io/kush-gupt/ansible-ed-25-03/ansible-ee:latest

# Run a simple command
podman run --rm ghcr.io/kush-gupt/ansible-ed-25-03/ansible-ee:latest ansible --version

# Run a playbook (mount your project directory)
podman run --rm \
  -v $(pwd):/runner \
  -v $(pwd)/inventory.ini:/runner/inventory \
  ghcr.io/kush-gupt/ansible-ed-25-03/ansible-ee:latest \
  ansible-playbook -i inventory playbooks/collect_artifacts.yml
```

### With Ansible Navigator (Recommended)

```bash
# Install ansible-navigator
pip install ansible-navigator

# Run a playbook
ansible-navigator run playbooks/collect_artifacts.yml \
  --execution-environment-image ghcr.io/kush-gupt/ansible-ed-25-03/ansible-ee:latest \
  --inventory inventory.ini

# Interactive mode
ansible-navigator \
  --execution-environment-image ghcr.io/kush-gupt/ansible-ed-25-03/ansible-ee:latest
```

### With Ansible Runner

```bash
# Install ansible-runner
pip install ansible-runner

# Create a project structure and run
ansible-runner run /path/to/project \
  --container-image ghcr.io/kush-gupt/ansible-ed-25-03/ansible-ee:latest \
  --playbook collect_artifacts.yml \
  --inventory inventory.ini
```

## Building Locally

To build the execution environment locally:

```bash
# Install ansible-builder
pip install ansible-builder

# Build the execution environment
ansible-builder build --file execution-environment.yml --tag local-ee:latest

# Test the build
podman run --rm local-ee:latest ansible --version
podman run --rm local-ee:latest ansible-galaxy collection list
```

## Automatic Builds

The execution environment is automatically built and pushed to GHCR when:

1. Changes are pushed to the `main` or `playbook-nest` branch that affect:
   - `execution-environment.yml`
   - `requirements.yml`
   - `requirements.txt`
   - `bindep.txt`
   - `_build/**`

2. The workflow is manually triggered via `workflow_dispatch`

3. A new release is created (triggered from the main collection workflow)

## Security

The execution environment build includes:

- **Security Scanning**: Trivy vulnerability scanning
- **Build Provenance**: SLSA build provenance attestations
- **Minimal Base**: Uses official Ansible base images
- **Dependency Pinning**: Specific version requirements where appropriate

## Configuration Files

The execution environment is defined by these files:

- `execution-environment.yml` - Main EE definition with embedded ansible.cfg
- `requirements.yml` - Ansible collection dependencies
- `requirements.txt` - Python package dependencies
- `bindep.txt` - System package dependencies

## Support

For issues with the execution environment:

1. Check the GitHub Actions workflow logs
2. Verify all dependency files are properly formatted
3. Test the build locally using `ansible-builder`
4. Review the security scan results in the GitHub Security tab
