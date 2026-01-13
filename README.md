# Learning yq

A comprehensive guide to mastering yq, the lightweight and portable command-line YAML processor.

## Overview

yq is a powerful command-line tool for processing YAML files, similar to how jq works for JSON. It enables you to query, create, modify, and validate YAML documents programmatically, making it essential for modern DevOps, GitOps, and cloud-native workflows.

## Why yq?

- **Cloud-Native Workflows**: Essential for Kubernetes, Crossplane, and GitOps
- **CI/CD Integration**: Automate configuration updates in pipelines
- **Infrastructure as Code**: Manipulate Terraform, Ansible, and cloud templates
- **Developer Productivity**: Quick YAML transformations without manual editing
- **Cross-Platform**: Works on macOS, Linux, and Windows

## Repository Structure

```
learning-yq/
├── README.md                          # This file
├── 01-installation/
│   ├── macos.md                      # macOS installation guide
│   ├── linux.md                      # Linux installation guide
│   ├── windows.md                    # Windows installation guide
│   └── vscode-integration.md         # VS Code setup
├── 02-basics/
│   ├── 01-reading-yaml.md            # Basic queries
│   ├── 02-creating-yaml.md           # Creating YAML from scratch
│   ├── 03-editing-yaml.md            # Modifying existing files
│   └── 04-data-types.md              # Working with different types
├── 03-advanced/
│   ├── 01-array-operations.md        # Array manipulation
│   ├── 02-conditional-logic.md       # Conditionals and filters
│   ├── 03-merging-files.md           # Combining YAML files
│   └── 04-operators.md               # Advanced operators
├── 04-kubernetes/
│   ├── 01-deployments.md             # Working with deployments
│   ├── 02-configmaps-secrets.md      # ConfigMaps and Secrets
│   └── 03-helm-values.md             # Helm values files
├── 05-crossplane/
│   ├── 01-compositions.md            # Crossplane Compositions
│   ├── 02-composite-resources.md    # XRDs and XRs
│   ├── 03-provider-configs.md       # Provider configurations
│   └── 04-patches.md                 # Patching strategies
├── 06-automation/
│   ├── 01-cicd-integration.md        # CI/CD pipeline examples
│   ├── 02-githooks.md                # Git hooks with yq
│   ├── 03-scripts.md                 # Automation scripts
│   └── 04-best-practices.md          # Best practices
├── 07-azure/
│   ├── 01-aks-manifests.md           # AKS-specific YAML
│   ├── 02-bicep-params.md            # Bicep parameter files
│   └── 03-azure-devops.md            # Azure DevOps integration
├── 08-examples/
│   ├── kubernetes/                    # K8s examples
│   ├── crossplane/                    # Crossplane examples
│   │   ├── azure-aks-composition.md  # AKS cluster composition
│   │   ├── azure-subscription-xrd.md # Subscription XRD
│   │   └── azure-subscription-composition.md # Subscription composition
│   ├── helm/                          # Helm examples
│   └── generic/                       # Generic YAML examples
└── 09-troubleshooting/
    ├── common-errors.md               # Common issues
    └── debugging.md                   # Debugging techniques
```

## Quick Start

### Installation

**macOS:**

```bash
brew install yq
```

**Linux:**

```bash
wget https://github.com/mikefarah/yq/releases/latest/download/yq_linux_amd64 -O /usr/local/bin/yq
chmod +x /usr/local/bin/yq
```

**Windows (PowerShell):**

```powershell
choco install yq
```

See [01-installation](./01-installation/) for detailed setup instructions.

### Basic Usage

**Read a value:**

```bash
yq '.metadata.name' deployment.yaml
```

**Update a value:**

```bash
yq -i '.spec.replicas = 5' deployment.yaml
```

**Create new YAML:**

```bash
yq -n '.apiVersion = "v1" | .kind = "ConfigMap"'
```

## Learning Path

### Beginner

1. [Installation guides](./01-installation/)
1. [Reading YAML](./02-basics/01-reading-yaml.md)
1. [Creating YAML](./02-basics/02-creating-yaml.md)
1. [Editing YAML](./02-basics/03-editing-yaml.md)

### Intermediate

1. [Array operations](./03-advanced/01-array-operations.md)
1. [Conditional logic](./03-advanced/02-conditional-logic.md)
1. [Kubernetes examples](./04-kubernetes/)
1. [Automation basics](./06-automation/)

### Advanced

1. [Crossplane manipulation](./05-crossplane/)
1. [Complex merging strategies](./03-advanced/03-merging-files.md)
1. [CI/CD integration](./06-automation/01-cicd-integration.md)
1. [Advanced scripting](./06-automation/03-scripts.md)

## Use Cases for Cloud Engineering

### Kubernetes & AKS

- Dynamic manifest generation
- ConfigMap and Secret management
- Multi-environment deployments
- Helm values customization

### Crossplane

- Composition template manipulation
- Provider configuration updates
- Composite Resource Definition editing
- Patch and transform operations

### GitOps

- Automated PR updates
- Configuration drift detection
- Multi-cluster deployments
- Environment promotion

### Azure DevOps

- Pipeline variable injection
- Dynamic task configuration
- Infrastructure parameter updates
- Release automation

## Key Features

- **JSONPath-like syntax**: Familiar querying for JSON/YAML
- **In-place editing**: Direct file modification with `-i`
- **Multiple file operations**: Merge, compare, and combine
- **Format conversion**: YAML ↔ JSON ↔ XML ↔ Properties
- **Scripting friendly**: Excellent for automation
- **Type preservation**: Maintains YAML types and formatting
- **Comments**: Can preserve YAML comments (v4+)

## Version Information

This guide covers yq v4 (mikefarah/yq), the most widely used version. Note that there are multiple tools named “yq” - this repository focuses on the Go-based implementation by Mike Farah.

**Verify your version:**

```bash
yq --version
```

Expected output: `yq (https://github.com/mikefarah/yq/) version v4.x.x`

## VS Code Integration

For optimal productivity, integrate yq with Visual Studio Code:

1. Install the yq extension
1. Configure YAML language server
1. Set up keyboard shortcuts
1. Create snippets for common operations

See [vscode-integration.md](./01-installation/vscode-integration.md) for complete setup.

## Real-World Scenarios

This repository includes practical examples for:

- **NATO/Atos IDP Project**: Backstage catalog manipulation, Crossplane templates
- **Azure Cloud Engineering**: AKS configuration, Azure resource YAML, subscription management
- **Azure Subscription Governance**: Enterprise subscription provisioning with policies, RBAC, and budgets
- **DevSecOps**: Security policy injection, compliance automation
- **Multi-Environment**: Dev/staging/production configuration management

## Contributing

This is a personal learning repository, but suggestions and improvements are welcome through issues and pull requests.

## Resources

### Official Documentation

- [yq GitHub Repository](https://github.com/mikefarah/yq)
- [Official Documentation](https://mikefarah.gitbook.io/yq/)

### Related Tools

- [jq](https://stedolan.github.io/jq/) - JSON processor
- [kubectl](https://kubernetes.io/docs/reference/kubectl/) - Kubernetes CLI
- [Crossplane](https://www.crossplane.io/) - Cloud-native control plane

### Complementary Learning

- [learning-kubernetes](https://github.com/vanHeemstraSystems/learning-kubernetes)
- [learning-crossplane](https://github.com/vanHeemstraSystems/learning-crossplane)
- [learning-azure](https://github.com/vanHeemstraSystems/learning-azure)

## License

This learning material is provided as-is for educational purposes.

## Author

Willem van Heemstra - Cloud Engineer | Security Domain Expert

-----

**Last Updated**: January 2026
**yq Version**: v4.40+
**Target Audience**: DevOps Engineers, Cloud Engineers, Platform Engineers
