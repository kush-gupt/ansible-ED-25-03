# UBI Micro Execution Environment - Minimal Attack Surface

This execution environment uses **Red Hat UBI 9 Micro** for the smallest possible Red Hat-based container image with minimal attack surface.

## Why UBI Micro?

- ✅ **Smallest Red Hat image**: ~40MB base (vs ~220MB for ubi9-minimal, ~250MB for ubi9/python)
- ✅ **No package manager**: Eliminates dnf/yum/rpm attack vectors (CVE-2021-35937, CVE-2021-35938, etc.)
- ✅ **Minimal components**: Only essential runtime libraries included
- ✅ **Red Hat support**: Official Red Hat Universal Base Image
- ✅ **Final image size**: ~140-180MB (vs ~250-300MB UBI Minimal, ~400-600MB standard UBI9)
- ✅ **Automated security testing**: CI/CD verifies no package managers or build tools present

## Security Benefits

### Attack Surface Reduction
- **No package managers**: `dnf`, `yum`, `rpm` not present in runtime
- **No build tools**: `gcc`, `make`, development headers excluded from final image
- **Minimal binaries**: Only `python3.11`, `git`, `ssh` in runtime
- **No shell utilities**: Reduced command injection risks
- **Non-root execution**: Container runs as UID 1000

### Compliance & Hardening
- Red Hat security-maintained base
- RHEL 9 security updates via UBI updates
- Minimal CVE exposure surface
- Suitable for air-gapped/restricted environments

## Architecture: Multi-Stage Build

```
┌─────────────────────────────────────┐
│  Stage 1: Builder                   │
│  (ubi9/python-311:9.4)             │
│                                     │
│  - Full package manager (microdnf) │
│  - Build tools (gcc, devel libs)   │
│  - Compile Python packages         │
│  - Install Ansible collections     │
│  - Size: ~800MB (discarded)        │
└─────────────┬───────────────────────┘
              │ COPY only runtime files
              ▼
┌─────────────────────────────────────┐
│  Stage 2: Runtime                   │
│  (ubi9/ubi-micro:9.4)              │
│                                     │
│  - Python 3.11 runtime only        │
│  - Pre-compiled Python packages    │
│  - Essential binaries (git, ssh)   │
│  - NO package manager              │
│  - NO build tools                  │
│  - Final size: ~140-180MB          │
└─────────────────────────────────────┘
```

## Build Instructions

### Prerequisites
```bash
# Podman is required (Docker can also work with minor adjustments)
podman --version

# Ensure you have the latest Containerfile.ubi-micro
git pull origin main
```

### Build the Image (Direct Podman Build)
```bash
cd /Users/kugupta/Documents/work/ansible-ED-25-03

# Build using the custom Containerfile.ubi-micro
podman build \
  -f Containerfile.ubi-micro \
  -t ghcr.io/kush-gupt/ansible-ed-25-03/ansible-ee:ubi-micro \
  --layers=true \
  --force-rm \
  .

# Build time: ~5-10 minutes (first build, cached after)
```

### Why Not ansible-builder?

ansible-builder doesn't support the multi-stage build pattern needed for UBI Micro. We use a custom `Containerfile.ubi-micro` that:
1. Builds everything in a `ubi9/python-311` builder stage (with package manager)
2. Copies only runtime files to `ubi9/ubi-micro` final stage (no package manager)

This is the **recommended and only supported method** for building this execution environment.

## Testing the Image

### 1. Verify Image Size
```bash
podman images | grep ansible-ee

# Expected output:
# ghcr.io/.../ansible-ee  ubi-micro  <id>  140-180MB
```

### 2. Verify No Package Manager (Security Check)
```bash
podman run --rm ghcr.io/kush-gupt/ansible-ed-25-03/ansible-ee:ubi-micro \
  which dnf yum rpm microdnf

# Expected: No output (commands not found) - GOOD!
```

### 3. Verify Ansible Functionality
```bash
# Check versions
podman run --rm ghcr.io/kush-gupt/ansible-ed-25-03/ansible-ee:ubi-micro \
  ansible --version

# Should show: ansible-core 2.17.x

# Check collections
podman run --rm ghcr.io/kush-gupt/ansible-ed-25-03/ansible-ee:ubi-micro \
  ansible-galaxy collection list

# Should show:
# cisco.asa         >= 6.0.0
# ansible.netcommon >= 8.0.0
```

### 4. Test ED 25-03 Playbook
```bash
# Mount your inventory and run a playbook
podman run --rm \
  -v $(pwd)/ansible:/runner:Z \
  -v $(pwd)/ansible/inventory.ini:/runner/inventory:Z \
  -e ANSIBLE_HOST_KEY_CHECKING=False \
  ghcr.io/kush-gupt/ansible-ed-25-03/ansible-ee:ubi-micro \
  ansible-playbook -i inventory \
    ansible_collections/cisa/ed_25_03/playbooks/assess_and_report.yml
```

### 5. Security Scan
```bash
# Scan for vulnerabilities with Trivy
trivy image ghcr.io/kush-gupt/ansible-ed-25-03/ansible-ee:ubi-micro

# Scan for vulnerabilities with Grype
grype ghcr.io/kush-gupt/ansible-ed-25-03/ansible-ee:ubi-micro

# Expected: Minimal/no HIGH or CRITICAL CVEs
```

## Troubleshooting

### Missing Shared Libraries
If you see errors like "error while loading shared libraries":

```bash
# Add the missing library to execution-environment.yml
# Find the library in the builder stage:
podman run --rm registry.access.redhat.com/ubi9/python-311:9.4 \
  ldd /usr/bin/<binary>

# Then add COPY statement for that .so file
```

### Collection Not Found
```bash
# Verify collections path
podman run --rm ghcr.io/.../ansible-ee:ubi-micro \
  ansible-galaxy collection list

# Check environment variable
podman run --rm ghcr.io/.../ansible-ee:ubi-micro \
  env | grep ANSIBLE_COLLECTIONS_PATH
```

### Permission Denied
```bash
# UBI Micro runs as non-root (UID 1000)
# Ensure mounted volumes have correct permissions:
chmod -R 755 ansible/
chown -R 1000:1000 ansible/artifacts/
```

## Comparison: Image Sizes

| Base Image | Final Size | Package Manager | Build Tools | Security Hardened |
|------------|-----------|----------------|-------------|-------------------|
| **ubi9/ubi-micro:9.4** | **~140-180MB** | ❌ No | ❌ No | ✅ Yes |
| ubi9-minimal:9.4 | ~250-300MB | ✅ microdnf | ❌ No | ⚠️ Partial |
| ubi9/python-311 | ~400-600MB | ✅ dnf | ⚠️ Optional | ❌ No |
| ubi9:9.4 | ~600-800MB | ✅ dnf/yum | ⚠️ Optional | ❌ No |

## Maintenance

### Updating the Image
```bash
# 1. Update base image version in execution-environment.yml
sed -i 's/:9.4/:9.5/g' execution-environment.yml

# 2. Update Python dependency versions in requirements.txt
# Check for security updates at https://pypi.org/

# 3. Rebuild
ansible-builder build --tag <new-tag>

# 4. Test thoroughly before deploying
```

### Security Patching
```bash
# UBI Micro receives security updates from Red Hat
# Rebuild monthly or when CVEs are announced:

ansible-builder build \
  --tag ghcr.io/kush-gupt/ansible-ed-25-03/ansible-ee:$(date +%Y%m)
```

## CI/CD Integration

Update your `.github/workflows/build-execution-environment.yml` to use the UBI Micro configuration:

```yaml
- name: Build UBI Micro EE
  run: |
    ansible-builder build \
      --tag ${{ env.REGISTRY }}/${{ github.repository }}/ansible-ee:ubi-micro-${{ github.sha }} \
      --tag ${{ env.REGISTRY }}/${{ github.repository }}/ansible-ee:ubi-micro-latest \
      --container-runtime podman
```

## References

- [Red Hat UBI Micro Documentation](https://www.redhat.com/en/blog/introduction-ubi-micro)
- [RHEL 9 Security Guide](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html/security_hardening/)
- [Ansible Builder Documentation](https://ansible-builder.readthedocs.io/)
- [UBI Images on Red Hat Catalog](https://catalog.redhat.com/software/containers/ubi9/ubi-micro/615bdf943f6014fa45ae1b58)

## Support

For issues specific to:
- **UBI Micro base**: Contact Red Hat Support or check RHEL documentation
- **Ansible Builder**: File issues at https://github.com/ansible/ansible-builder
- **ED 25-03 Collection**: File issues at your repository

---

**Last Updated**: 2025-09-29  
**UBI Micro Version**: 9.6+  
**Ansible Core**: 2.17.x
