# Reading YAML with yq

## Overview

Reading and querying YAML files is the fundamental operation in yq. This guide covers all the essential patterns for extracting data from YAML documents.

## Basic Syntax

The basic syntax for reading values:

```bash
yq 'EXPRESSION' file.yaml
```

## Simple Value Extraction

### Reading Top-Level Values

**Example file (config.yaml):**

```yaml
name: my-app
version: 1.0.0
enabled: true
```

**Read values:**

```bash
yq '.name' config.yaml
# Output: my-app

yq '.version' config.yaml
# Output: 1.0.0

yq '.enabled' config.yaml
# Output: true
```

### Reading Nested Values

**Example file (service.yaml):**

```yaml
metadata:
  name: web-service
  namespace: production
spec:
  port: 8080
  protocol: HTTP
```

**Read nested values:**

```bash
yq '.metadata.name' service.yaml
# Output: web-service

yq '.spec.port' service.yaml
# Output: 8080
```

## Working with Arrays

### Array Index Access

**Example file (deployment.yaml):**

```yaml
spec:
  containers:
  - name: app
    image: nginx:1.21
  - name: sidecar
    image: busybox:latest
```

**Access by index:**

```bash
# First container
yq '.spec.containers[0]' deployment.yaml

# First container name
yq '.spec.containers[0].name' deployment.yaml
# Output: app

# All container names
yq '.spec.containers[].name' deployment.yaml
# Output:
# app
# sidecar
```

### Array Length

```bash
yq '.spec.containers | length' deployment.yaml
# Output: 2
```

## Filtering and Selection

### Select by Condition

**Find container by name:**

```bash
yq '.spec.containers[] | select(.name == "app")' deployment.yaml
```

**Find containers with specific image:**

```bash
yq '.spec.containers[] | select(.image | test("nginx"))' deployment.yaml
```

### Multiple Conditions

```bash
yq '.spec.containers[] | select(.name == "app" and .image | test("nginx"))' deployment.yaml
```

## Output Formats

### Default YAML Output

```bash
yq '.metadata' service.yaml
# Output in YAML format
```

### JSON Output

```bash
yq -o json '.metadata' service.yaml
# Output: {"name":"web-service","namespace":"production"}
```

### Raw Output (No Quotes)

```bash
yq -r '.metadata.name' service.yaml
# Output: web-service (no quotes)
```

### Pretty Print

```bash
yq -P '.' service.yaml
# Formats with proper indentation
```

## Multiple Values

### Read Multiple Fields

**Using comma operator:**

```bash
yq '.metadata.name, .metadata.namespace' service.yaml
# Output:
# web-service
# production
```

**Create JSON object:**

```bash
yq -o json '{name: .metadata.name, ns: .metadata.namespace}' service.yaml
# Output: {"name":"web-service","ns":"production"}
```

## Working with Null Values

### Check for Null

```bash
yq '.nonexistent' service.yaml
# Output: null

yq -e '.nonexistent' service.yaml
# Exits with error if null
```

### Provide Defaults

```bash
yq '.nonexistent // "default-value"' service.yaml
# Output: default-value
```

## Kubernetes-Specific Examples

### Read Pod Information

```bash
# Get pod name
yq '.metadata.name' pod.yaml

# Get all container images
yq '.spec.containers[].image' pod.yaml

# Get resource requests
yq '.spec.containers[0].resources.requests.cpu' pod.yaml
```

### Read Deployment Details

```bash
# Get replica count
yq '.spec.replicas' deployment.yaml

# Get deployment strategy
yq '.spec.strategy.type' deployment.yaml

# Get all environment variables
yq '.spec.template.spec.containers[0].env[]' deployment.yaml
```

## Crossplane Examples

### Read Composition Info

```bash
# Get composition name
yq '.metadata.name' composition.yaml

# Get composite type
yq '.spec.compositeTypeRef.kind' composition.yaml

# List all resource names
yq '.spec.resources[].name' composition.yaml

# Get provider for each resource
yq '.spec.resources[] | {name: .name, kind: .base.kind}' composition.yaml
```

## Common Patterns

### Check if Key Exists

```bash
yq 'has("metadata")' service.yaml
# Output: true

yq '.metadata | has("labels")' service.yaml
# Output: true/false
```

### Get All Keys

```bash
yq '.metadata | keys' service.yaml
# Output: [name, namespace]
```

### Type Checking

```bash
yq '.spec.port | type' service.yaml
# Output: !!int

yq '.metadata.name | type' service.yaml
# Output: !!str
```

## Reading Multiple Files

### Process Multiple Files

```bash
# Read same path from multiple files
yq '.metadata.name' file1.yaml file2.yaml file3.yaml

# Merge multiple files
yq eval-all '.' file1.yaml file2.yaml
```

### Compare Values Across Files

```bash
# Get versions from all files
for file in *.yaml; do
  echo "$file: $(yq '.version' "$file")"
done
```

## Practical Use Cases

### Extract Environment Variables

```bash
# From deployment to shell export format
yq '.spec.template.spec.containers[0].env[] | "export " + .name + "=" + .value' deployment.yaml
```

### Generate Documentation

```bash
# Extract all annotations
yq '.metadata.annotations | to_entries | .[] | "- " + .key + ": " + .value' resource.yaml
```

### Audit Resource Configuration

```bash
# Check if resources have limits
yq '.spec.containers[] | select(.resources.limits == null) | .name' deployment.yaml
```

## Troubleshooting

### Common Errors

**Error: null value**

```bash
# Bad
yq '.nonexistent.field' config.yaml
# Error: field not found

# Good - use optional access
yq '.nonexistent.field // "not found"' config.yaml
```

**Error: array index out of bounds**

```bash
# Bad
yq '.spec.containers[10]' deployment.yaml
# Error: index out of range

# Good - check length first
yq 'if (.spec.containers | length) > 10 then .spec.containers[10] else "not found" end' deployment.yaml
```

## Performance Tips

### Avoid Multiple yq Calls

**Inefficient:**

```bash
name=$(yq '.metadata.name' file.yaml)
namespace=$(yq '.metadata.namespace' file.yaml)
version=$(yq '.metadata.labels.version' file.yaml)
```

**Efficient:**

```bash
read name namespace version < <(yq -o json '{
  name: .metadata.name,
  namespace: .metadata.namespace,
  version: .metadata.labels.version
}' file.yaml | jq -r '.name, .namespace, .version')
```

## Next Steps

- [Creating YAML](./02-creating-yaml.md)
- [Editing YAML](./03-editing-yaml.md)
- [Advanced Operations](../03-advanced/)

## Quick Reference

```bash
# Basic read
yq '.path.to.value' file.yaml

# Array access
yq '.array[0]' file.yaml
yq '.array[]' file.yaml           # All elements

# Select/filter
yq '.array[] | select(.key == "value")' file.yaml

# Output formats
yq -o json '.' file.yaml          # JSON
yq -o yaml '.' file.yaml          # YAML (default)
yq -r '.value' file.yaml          # Raw (no quotes)

# Multiple values
yq '.key1, .key2' file.yaml

# Default values
yq '.key // "default"' file.yaml

# Type checking
yq '.key | type' file.yaml

# Check existence
yq 'has("key")' file.yaml
```
