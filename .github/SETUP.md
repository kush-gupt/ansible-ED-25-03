# GitHub Actions Setup for Ansible Collection

This document explains how to configure the GitHub Actions pipeline for building and publishing the CISA ED 25-03 Ansible collection.

## Required GitHub Secrets

To enable automatic publishing to Ansible Galaxy, you need to configure the following repository secrets:

### 1. ANSIBLE_GALAXY_API_KEY

This is your Ansible Galaxy API token required for publishing collections.

**How to get your API key:**
1. Go to [Ansible Galaxy](https://galaxy.ansible.com/)
2. Log in with your account
3. Click on your profile (top right) → "My Content" → "API Key"
4. Copy the API key

**How to add the secret to GitHub:**
1. Go to your repository on GitHub
2. Navigate to Settings → Secrets and variables → Actions
3. Click "New repository secret"
4. Name: `ANSIBLE_GALAXY_API_KEY`
5. Value: Paste your Ansible Galaxy API key
6. Click "Add secret"

## Workflow Triggers

The GitHub Actions workflow is triggered by:

- **Push to main branch**: Builds, tests, and publishes the collection
- **Pull requests to main**: Only builds and tests (no publishing)
- **Git tags starting with 'v'**: Builds, publishes, and creates a GitHub release

## Workflow Jobs

### 1. lint-and-test
- Runs ansible-lint and yamllint
- Validates galaxy.yml structure
- Installs and checks collection dependencies

### 2. build
- Builds the Ansible collection tarball
- Uploads the artifact for use by other jobs
- Extracts version from galaxy.yml

### 3. publish-galaxy
- Only runs on main branch pushes or version tags
- Downloads the built collection artifact
- Publishes to Ansible Galaxy using the API key
- Verifies successful publication

### 4. trigger-execution-environment
- Only runs on main branch pushes
- Triggers the execution environment build workflow
- Builds and pushes the Ansible execution environment to GHCR

### 5. create-release
- Only runs for version tags (e.g., v0.1.0)
- Creates a GitHub release with the collection tarball
- Auto-generates release notes
- Also triggers execution environment build for the release

## Version Management

The workflow automatically extracts the version from `galaxy.yml`. To release a new version:

1. Update the version in `ansible/ansible_collections/cisa/ed_25_03/galaxy.yml`
2. Commit and push to main (this will publish to Galaxy)
3. Create a git tag for GitHub releases:
   ```bash
   git tag v0.1.1
   git push origin v0.1.1
   ```

## Collection Dependencies

Make sure your `ansible/requirements.yml` file exists and contains all necessary collection dependencies:

```yaml
---
collections:
  - name: ansible.netcommon
    version: ">=6.0.0"
  - name: cisco.asa
    version: ">=5.0.0"
```

## Troubleshooting

### Build Failures
- Check that `galaxy.yml` has all required fields
- Ensure collection dependencies are available
- Verify ansible-lint passes locally

### Publishing Failures
- Verify `ANSIBLE_GALAXY_API_KEY` secret is set correctly
- Check that the namespace/collection name doesn't conflict
- Ensure you have publishing permissions for the namespace

### Release Creation Failures
- Verify the repository has "Contents: write" permissions
- Check that the tag follows the expected format (v*)

## Execution Environment

The repository also builds an Ansible execution environment (EE) that includes all the necessary dependencies to run the CISA ED 25-03 collection. The execution environment is automatically built and pushed to GitHub Container Registry (GHCR) when changes are made to the main branch.

### Execution Environment Workflow

The `build-execution-environment.yml` workflow:
- Builds an Ansible execution environment using `ansible-builder`
- Includes all collection dependencies (cisco.asa, ansible.netcommon)
- Runs security scanning with Trivy
- Pushes the image to GHCR at `ghcr.io/[owner]/[repo]/ansible-ee`
- Generates build provenance attestations

### Using the Execution Environment

You can use the execution environment with various tools:

```bash
# Pull the latest execution environment
podman pull ghcr.io/[owner]/[repo]/ansible-ee:latest

# Use with ansible-navigator (recommended)
ansible-navigator run playbook.yml \
  --execution-environment-image ghcr.io/[owner]/[repo]/ansible-ee:latest

# Run a playbook using the execution environment directly
podman run --rm -v $(pwd):/runner ghcr.io/[owner]/[repo]/ansible-ee:latest ansible-playbook -i inventory.ini playbook.yml
```
## Local Testing

Before pushing, you can test the collection build locally:

```bash
# Navigate to collection directory
cd ansible/ansible_collections/cisa/ed_25_03

# Build the collection
ansible-galaxy collection build

# Test installation
ansible-galaxy collection install kush_gupt-ed_25_03-*.tar.gz --force
```

### Testing Execution Environment Locally

You can also test the execution environment build locally:

```bash
# Install ansible-builder
pip install ansible-builder

# Build the execution environment
ansible-builder build --file execution-environment.yml --tag local-ee:latest

# Test the execution environment
podman run --rm local-ee:latest ansible --version
podman run --rm local-ee:latest ansible-galaxy collection list
```
