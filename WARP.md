# WARP.md

This file provides guidance to WARP (warp.dev) when working with code in this repository.

## Project Overview

This repository provides opinionated examples for building VMware vSphere virtual machine images using HashiCorp Packer and the Packer Plugin for VMware vSphere (`vsphere-iso` builder). All examples use HashiCorp Configuration Language (HCL).

## Dependencies

Required tools (with minimum versions defined in `project.json`):
- **packer** >= 1.12.0
- **ansible** >= 2.17.0
- **jq** (required)
- **git** (required)
- **gomplate** (for CI pipeline generation)
- **terraform** (optional, for testing)

Check dependencies:
```shell
./build.sh --deps
```

## Common Build Commands

### Interactive Build (Recommended)
```shell
./build.sh
```
This launches an interactive menu to select OS, distribution, version, and edition.

### Non-Interactive Build
```shell
./build.sh --os "Linux" --dist "Ubuntu Server" --version "24.04 LTS" --auto-continue
./build.sh --os "Windows" --dist "Windows Server" --version "2022" --edition "Standard" --auto-continue
```

### Direct Packer Build
For building a specific source (e.g., Windows Server 2022 Standard Core):

```shell
# Initialize plugins
packer init builds/windows/server/2022/.

# Build specific image
packer build -force -on-error=ask \
    --only vsphere-iso.windows-server-standard-core \
    -var-file="config/build.pkrvars.hcl" \
    -var-file="config/common.pkrvars.hcl" \
    -var-file="config/vsphere.pkrvars.hcl" \
    builds/windows/server/2022
```

## Configuration Setup

### Initialize Configuration Files
```shell
./config.sh [config_path]
```
This copies `.pkrvars.hcl.example` files to the `config/` directory (or specified path) and removes the `.example` extension. Existing configurations are backed up with timestamps.

### Environment Variables (Alternative to Config Files)
```shell
. ./set-envvars.sh
```
**Note:** Must be run with source (`.`) to export variables to current shell. Environment variables take precedence over configuration files.

## ISO Management

### Download ISOs
```shell
./download.sh
```
Interactive menu to download guest OS ISOs with checksum verification.

### Set Offline Token (Red Hat)
```shell
export rhsm_offline_token="your_token_value"
```

## Architecture

### Directory Structure

- **`builds/`** - Packer build definitions organized by OS
  - **`linux/`** - Linux distributions (almalinux, centos, debian, fedora, oracle, photon, rhel, rocky, sles, ubuntu)
  - **`windows/`** - Windows editions (desktop, server)
  - **`*.pkrvars.hcl.example`** - Global configuration templates (vsphere, common, build, ansible, proxy, network, etc.)
  
- **`builds/[os]/[distribution]/[version]/`** - Each OS variant contains:
  - `*.pkr.hcl` - Packer build files (sources, variables, build blocks)
  - `data/` - Boot configuration templates (user-data, meta-data, network, storage, autounattend.pkrtpl.hcl)
  - `*.pkrvars.hcl.example` - OS-specific variable overrides

- **`ansible/`** - Provisioning playbooks
  - `linux-playbook.yml` / `windows-playbook.yml`
  - `*-requirements.yml` - Ansible Galaxy role dependencies
  - `roles/` - Custom Ansible roles

- **`terraform/`** - Testing examples
  - `vsphere-role/` - vSphere role creation
  - `vsphere-virtual-machine/` - VM deployment from built images

- **`config/`** - Generated configuration directory (gitignored)
  - Created by `config.sh`
  - Contains actual `.pkrvars.hcl` files with credentials/settings

- **`scripts/`** - Helper scripts for builds

- **`docs/`** - MkDocs documentation source

- **`project.json`** - Central metadata file defining all OS distributions, versions, download links, checksums, and build file mappings

### Build Flow

1. **Configuration** - Variables loaded from multiple `.pkrvars.hcl` files (vsphere, common, build, ansible, network, storage, proxy, rhsm/scc)
2. **Source Definition** - `vsphere-iso` builder in `*.pkr.hcl` files defines VM settings, boot commands, and provisioners
3. **Boot Configuration** - Templates in `data/` directory provide automated installation configs (cloud-init for Linux, autounattend.xml for Windows)
4. **Provisioning** - Ansible playbooks run post-installation customization
5. **Artifact** - By default, images are exported as OVF templates to vSphere Content Library and temporary VMs are destroyed

### Key Patterns

- **Variable Hierarchy**: Global defaults → OS-specific → User config files
- **Template Generation**: `*.pkrtpl.hcl` files use Packer templating for dynamic boot configs
- **Data Sources**: Supports both HTTP server (`http`) and attached CD (`disk`) for boot configs via `common_data_source` variable
- **Content Library**: Default artifact destination; can also export to OVF files or convert to templates
- **Version Tracking**: Uses git commit hash for build versioning via `git-repository` data source

## Testing

The `tests/` directory contains validation scripts for different configurations (network, storage). Tests use Packer directly with specific variable files.

## Documentation

### Build Local Docs
```shell
make docs-install  # Install dependencies
make docs-serve    # Serve docs at http://127.0.0.1:8000
make docs-build    # Build static docs
make docs-uninstall
```

### Update GitLab CI
```shell
make update-gitlab-ci  # Regenerates .gitlab-ci.yml from build-ci.tmpl using gomplate
```

## Important Notes

- **Credentials**: Never commit credentials in `.pkrvars.hcl` files. Use environment variables or keep config files gitignored.
- **Content Library**: Default behavior exports to vSphere Content Library as OVF. Encrypted VMs (e.g., Windows 11 with vTPM) cannot be cloned to content library as OVF templates.
- **Build Script Options**: Use `--show` to display the actual `packer build` command without executing it.
- **Dependencies Check**: Always run `./build.sh --deps` before first build to verify all required tools are installed.

## Contributing

- Use Conventional Commits format for commit messages
- All commits must be signed-off for Developer Certificate of Origin (DCO)
- Work against the `develop` branch
- Open discussions for significant changes before submitting PRs
- Link PRs to related issues

Example commit:
```shell
git commit --signoff --message "feat: add support for x

Added support for x.

Signed-off-by: Jane Doe <jdoe@example.com>

Ref: #123"
```
