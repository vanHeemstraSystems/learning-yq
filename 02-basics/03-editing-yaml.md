# Editing YAML with yq

## Overview

Editing existing YAML files is one of yq’s most powerful features. This guide covers in-place editing, selective updates, and safe modification patterns.

## In-Place Editing

### Basic In-Place Edit

The `-i` flag modifies files directly without creating backups.

```bash
# Update a single value
yq -i '.version = "2.0"' config.yaml

# Update nested value
yq -i '.metadata.namespace = "production"' deployment.yaml

# Update multiple values
yq -i '
  .version = "2.0" |
  .enabled = true |
  .replicas = 5
' config.yaml
```

### Safe In-Place Editing

Always validate before making changes:

```bash
# Create backup first
cp config.yaml config.yaml.backup

# Make changes
yq -i '.version = "2.0"' config.yaml

# Validate result
if yq '.' config.yaml > /dev/null 2>&1; then
  echo "Edit successful"
  rm config.yaml.backup
else
  echo "Edit failed, restoring backup"
  mv config.yaml.backup config.yaml
fi
```

## Updating Values

### Update Simple Values

```bash
# String
yq -i '.name = "new-name"' config.yaml

# Number
yq -i '.port = 8080' config.yaml

# Boolean
yq -i '.enabled = true' config.yaml

# Null
yq -i '.optional = null' config.yaml
```

### Update Nested Values

```bash
# Update nested field
yq -i '.metadata.labels.environment = "production"' deployment.yaml

# Update deeply nested
yq -i '.spec.template.spec.containers[0].image = "nginx:1.22"' deployment.yaml

# Create path if it doesn't exist
yq -i '.new.nested.path = "value"' config.yaml
```

### Update Using Existing Values

```bash
# Increment a value
yq -i '.spec.replicas = .spec.replicas + 1' deployment.yaml

# Concatenate strings
yq -i '.metadata.name = .metadata.name + "-prod"' config.yaml

# Reference other fields
yq -i '.spec.serviceName = .metadata.name' deployment.yaml
```

## Working with Arrays

### Update Array Elements

```bash
# Update specific index
yq -i '.items[0] = "new-first-item"' config.yaml

# Update last element
yq -i '.items[-1] = "new-last-item"' config.yaml

# Update all elements
yq -i '.items[] = "same-value"' config.yaml
```

### Add to Arrays

```bash
# Append to array
yq -i '.items += "new-item"' config.yaml

# Append multiple items
yq -i '.items += ["item1", "item2", "item3"]' config.yaml

# Prepend to array
yq -i '.items = ["first"] + .items' config.yaml

# Insert at specific position
yq -i '.items = .items[0:2] + ["inserted"] + .items[2:]' config.yaml
```

### Remove from Arrays

```bash
# Remove by index
yq -i 'del(.items[0])' config.yaml

# Remove last element
yq -i 'del(.items[-1])' config.yaml

# Remove by value
yq -i '.items = .items | filter(. != "unwanted")' config.yaml

# Remove duplicates
yq -i '.items |= unique' config.yaml
```

### Update Array Objects

```bash
# Update specific object in array by index
yq -i '.containers[0].image = "nginx:1.22"' deployment.yaml

# Update by condition
yq -i '
  (.containers[] | select(.name == "app")).image = "nginx:1.22"
' deployment.yaml

# Update multiple fields in array object
yq -i '
  (.containers[] | select(.name == "app")) |= (
    .image = "nginx:1.22" |
    .imagePullPolicy = "Always"
  )
' deployment.yaml
```

## Conditional Updates

### Update if Field Exists

```bash
# Only update if field exists
yq -i '
  if has("version") then
    .version = "2.0"
  else
    .
  end
' config.yaml

# Update with default if missing
yq -i '.version = (.version // "1.0")' config.yaml
```

### Update Based on Value

```bash
# Update if value matches
yq -i '
  if .environment == "dev" then
    .replicas = 1
  elif .environment == "staging" then
    .replicas = 2
  else
    .replicas = 3
  end
' config.yaml

# Update multiple fields conditionally
yq -i '
  if .spec.replicas < 3 then
    .spec.replicas = 3 |
    .metadata.annotations.scaled = "true"
  else
    .
  end
' deployment.yaml
```

### Update Matching Resources

```bash
# Update all containers with specific name
yq -i '
  (.spec.containers[] | select(.name == "app")).image = "nginx:1.22"
' deployment.yaml

# Update all services in specific namespace
yq -i '
  (select(.metadata.namespace == "production") | .spec.type) = "LoadBalancer"
' service.yaml
```

## Kubernetes-Specific Edits

### Update Container Image

```bash
# Update first container
yq -i '.spec.template.spec.containers[0].image = "nginx:1.22"' deployment.yaml

# Update by container name
yq -i '
  (.spec.template.spec.containers[] | select(.name == "app")).image = "nginx:1.22"
' deployment.yaml

# Update image tag only
yq -i '
  (.spec.template.spec.containers[0].image) |= sub(":.*", ":1.22")
' deployment.yaml
```

### Update Replicas

```bash
# Set replicas
yq -i '.spec.replicas = 5' deployment.yaml

# Increase replicas
yq -i '.spec.replicas = .spec.replicas + 2' deployment.yaml

# Scale down to zero
yq -i '.spec.replicas = 0' deployment.yaml
```

### Update Resource Limits

```bash
# Update CPU limits
yq -i '
  .spec.template.spec.containers[0].resources.limits.cpu = "500m" |
  .spec.template.spec.containers[0].resources.requests.cpu = "250m"
' deployment.yaml

# Update memory
yq -i '
  .spec.template.spec.containers[0].resources.limits.memory = "512Mi" |
  .spec.template.spec.containers[0].resources.requests.memory = "256Mi"
' deployment.yaml
```

### Update Environment Variables

```bash
# Add environment variable
yq -i '
  .spec.template.spec.containers[0].env += {
    "name": "NEW_VAR",
    "value": "new-value"
  }
' deployment.yaml

# Update existing env var
yq -i '
  (.spec.template.spec.containers[0].env[] | select(.name == "LOG_LEVEL")).value = "debug"
' deployment.yaml

# Remove env var
yq -i '
  .spec.template.spec.containers[0].env = [
    .spec.template.spec.containers[0].env[] | select(.name != "UNWANTED_VAR")
  ]
' deployment.yaml
```

### Update Labels and Annotations

```bash
# Add label
yq -i '.metadata.labels.version = "v2"' deployment.yaml

# Update multiple labels
yq -i '
  .metadata.labels.version = "v2" |
  .metadata.labels.team = "platform"
' deployment.yaml

# Add annotation
yq -i '.metadata.annotations."deployment.kubernetes.io/revision" = "5"' deployment.yaml

# Remove label
yq -i 'del(.metadata.labels.old-label)' deployment.yaml
```

### Update Service Configuration

```bash
# Change service type
yq -i '.spec.type = "LoadBalancer"' service.yaml

# Update port
yq -i '.spec.ports[0].port = 443' service.yaml

# Add new port
yq -i '
  .spec.ports += {
    "name": "metrics",
    "port": 9090,
    "targetPort": 9090,
    "protocol": "TCP"
  }
' service.yaml
```

## Crossplane-Specific Edits

### Update Composition Resources

```bash
# Update provider config for all resources
yq -i '
  .spec.resources[].base.spec.providerConfigRef.name = "azure-provider-prod"
' composition.yaml

# Update specific resource
yq -i '
  (.spec.resources[] | select(.name == "database")).base.spec.forProvider.tier = "Premium"
' composition.yaml

# Update location for all Azure resources
yq -i '
  (.spec.resources[].base.spec.forProvider.location) = "northeurope"
' composition.yaml
```

### Update Patches

```bash
# Add patch to resource
yq -i '
  (.spec.resources[] | select(.name == "database")).patches += [{
    "type": "FromCompositeFieldPath",
    "fromFieldPath": "spec.parameters.region",
    "toFieldPath": "spec.forProvider.location"
  }]
' composition.yaml

# Update patch transform
yq -i '
  (.spec.resources[0].patches[] | 
   select(.fromFieldPath == "spec.parameters.size")).transforms[0].map.large = "Standard_D8s_v3"
' composition.yaml
```

### Update Composite Resource Claims

```bash
# Update claim parameters
yq -i '.spec.parameters.nodeCount = 5' claim.yaml

# Update composition selector
yq -i '.spec.compositionSelector.matchLabels.environment = "production"' claim.yaml

# Change provider config
yq -i '.spec.resourceConfig.providerConfigName = "azure-provider-prod"' claim.yaml
```

## Advanced Editing Patterns

### Merge Updates

```bash
# Merge new data into existing
yq -i '. *= {
  "metadata": {
    "labels": {
      "new-label": "value"
    }
  }
}' config.yaml

# Deep merge with override
yq -i '. *+ {
  "spec": {
    "replicas": 5,
    "strategy": {
      "type": "RollingUpdate"
    }
  }
}' deployment.yaml
```

### Batch Updates

```bash
# Update multiple files
for file in deployments/*.yaml; do
  yq -i '.spec.replicas = 3' "$file"
done

# Update with different values per file
for env in dev staging prod; do
  yq -i ".metadata.labels.environment = \"$env\"" "config-$env.yaml"
done
```

### Selective Field Updates

```bash
# Update only if value matches pattern
yq -i '
  (.spec.template.spec.containers[] | 
   select(.image | test("nginx"))).image |= sub(":.*", ":1.22")
' deployment.yaml

# Update based on label
yq -i '
  (select(.metadata.labels.app == "web") | .spec.replicas) = 5
' deployment.yaml
```

### Transform While Updating

```bash
# Convert to uppercase
yq -i '.metadata.name |= upcase' config.yaml

# Convert to lowercase
yq -i '.metadata.namespace |= downcase' config.yaml

# Replace characters
yq -i '.metadata.name |= sub("_", "-")' config.yaml

# Add prefix
yq -i '.metadata.name = "prod-" + .metadata.name' config.yaml
```

## Deletion Operations

### Delete Fields

```bash
# Delete single field
yq -i 'del(.unwanted)' config.yaml

# Delete nested field
yq -i 'del(.spec.unwanted.field)' config.yaml

# Delete multiple fields
yq -i 'del(.field1, .field2, .field3)' config.yaml
```

### Delete Array Elements

```bash
# Delete by index
yq -i 'del(.items[0])' config.yaml

# Delete by condition
yq -i 'del(.items[] | select(.status == "inactive"))' config.yaml

# Delete all matching
yq -i 'del(.containers[] | select(.name == "sidecar"))' deployment.yaml
```

### Delete Labels/Annotations

```bash
# Delete specific label
yq -i 'del(.metadata.labels.old-label)' deployment.yaml

# Delete all labels matching pattern
yq -i 'del(.metadata.labels | to_entries[] | select(.key | test("temp-")))' deployment.yaml

# Delete annotation
yq -i 'del(.metadata.annotations."deprecated-annotation")' deployment.yaml
```

## Automation Scripts

### Safe Batch Edit Script

```bash
#!/bin/bash
# safe-batch-edit.sh

FILES="$1"
EXPRESSION="$2"

for file in $FILES; do
  echo "Processing $file..."
  
  # Create backup
  cp "$file" "${file}.backup"
  
  # Apply edit
  if yq -i "$EXPRESSION" "$file" 2>&1; then
    # Validate result
    if yq '.' "$file" > /dev/null 2>&1; then
      echo "✓ Successfully updated $file"
      rm "${file}.backup"
    else
      echo "✗ Invalid YAML after edit, restoring $file"
      mv "${file}.backup" "$file"
    fi
  else
    echo "✗ Edit failed for $file, restoring"
    mv "${file}.backup" "$file"
  fi
done
```

Usage:

```bash
./safe-batch-edit.sh "configs/*.yaml" '.version = "2.0"'
```

### Environment-Specific Update

```bash
#!/bin/bash
# update-for-environment.sh

ENVIRONMENT="$1"
FILES="$2"

case $ENVIRONMENT in
  dev)
    REPLICAS=1
    CPU="100m"
    MEMORY="128Mi"
    ;;
  staging)
    REPLICAS=2
    CPU="200m"
    MEMORY="256Mi"
    ;;
  prod)
    REPLICAS=5
    CPU="500m"
    MEMORY="512Mi"
    ;;
  *)
    echo "Unknown environment: $ENVIRONMENT"
    exit 1
    ;;
esac

for file in $FILES; do
  yq -i "
    .spec.replicas = $REPLICAS |
    .spec.template.spec.containers[0].resources.requests.cpu = \"$CPU\" |
    .spec.template.spec.containers[0].resources.requests.memory = \"$MEMORY\" |
    .metadata.labels.environment = \"$ENVIRONMENT\"
  " "$file"
  echo "Updated $file for $ENVIRONMENT"
done
```

Usage:

```bash
./update-for-environment.sh prod "deployments/*.yaml"
```

### Rollback Changes

```bash
#!/bin/bash
# rollback.sh

if [ ! -d ".backup" ]; then
  echo "No backup directory found"
  exit 1
fi

for backup in .backup/*.backup; do
  original="${backup%.backup}"
  original="${original#.backup/}"
  
  if [ -f "$original" ]; then
    echo "Rolling back $original"
    mv "$backup" "$original"
  fi
done

rmdir .backup 2>/dev/null
echo "Rollback complete"
```

## Best Practices

### 1. Always Validate After Editing

```bash
# Edit with validation
yq -i '.version = "2.0"' config.yaml && \
yq '.' config.yaml > /dev/null && \
echo "Valid" || echo "Invalid"
```

### 2. Use Backups for Important Changes

```bash
# Create timestamped backup
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
cp config.yaml "config.yaml.${TIMESTAMP}.backup"
yq -i '.version = "2.0"' config.yaml
```

### 3. Test Changes on Copy First

```bash
# Test on copy
cp deployment.yaml deployment.yaml.test
yq -i '.spec.replicas = 10' deployment.yaml.test

# Verify it worked
if yq '.' deployment.yaml.test > /dev/null 2>&1; then
  yq -i '.spec.replicas = 10' deployment.yaml
  rm deployment.yaml.test
fi
```

### 4. Use Atomic Operations

```bash
# Instead of multiple separate edits
yq -i '.field1 = "value1"' config.yaml
yq -i '.field2 = "value2"' config.yaml
yq -i '.field3 = "value3"' config.yaml

# Do single atomic edit
yq -i '
  .field1 = "value1" |
  .field2 = "value2" |
  .field3 = "value3"
' config.yaml
```

### 5. Use Version Control

```bash
# Before making changes
git add config.yaml
git commit -m "Before yq changes"

# Make changes
yq -i '.version = "2.0"' config.yaml

# Review changes
git diff config.yaml

# If satisfied
git add config.yaml
git commit -m "Updated version to 2.0"

# If not satisfied
git checkout config.yaml
```

## Common Patterns

### Update Image Tags in Multiple Deployments

```bash
NEW_TAG="v2.1.0"

for deployment in k8s/deployments/*.yaml; do
  yq -i "
    (.spec.template.spec.containers[] | 
     select(.image | test(\"myapp\")).image) |= 
    sub(\":.*\", \":$NEW_TAG\")
  " "$deployment"
done
```

### Update Namespace in All Resources

```bash
NEW_NAMESPACE="production"

for file in k8s/*.yaml; do
  yq -i ".metadata.namespace = \"$NEW_NAMESPACE\"" "$file"
done
```

### Add Common Labels

```bash
COMMON_LABELS='{"team": "platform", "managed-by": "gitops"}'

for file in k8s/*.yaml; do
  yq -i ".metadata.labels *= $COMMON_LABELS" "$file"
done
```

## Troubleshooting

### Issue: Changes Not Applied

```bash
# Check if file is writable
ls -l config.yaml

# Verify yq syntax
yq -i '.field = "value"' config.yaml  # Correct
yq -i .field = "value" config.yaml    # Wrong (missing quotes)
```

### Issue: File Corrupted After Edit

```bash
# Always have backup
cp config.yaml config.yaml.backup
yq -i '.field = "value"' config.yaml

# Validate
if ! yq '.' config.yaml > /dev/null 2>&1; then
  echo "Corrupted, restoring backup"
  mv config.yaml.backup config.yaml
fi
```

### Issue: Selective Update Not Working

```bash
# Ensure selector matches
yq '.containers[] | select(.name == "app")' deployment.yaml

# If nothing returned, check actual names
yq '.containers[].name' deployment.yaml
```

## Next Steps

- [Working with Data Types](./04-data-types.md)
- [Array Operations](../03-advanced/01-array-operations.md)
- [Conditional Logic](../03-advanced/02-conditional-logic.md)

## Quick Reference

```bash
# Basic edit
yq -i '.key = "value"' file.yaml

# Multiple fields
yq -i '.key1 = "val1" | .key2 = "val2"' file.yaml

# Update nested
yq -i '.parent.child = "value"' file.yaml

# Array element
yq -i '.items[0] = "value"' file.yaml

# Add to array
yq -i '.items += "new"' file.yaml

# Delete field
yq -i 'del(.unwanted)' file.yaml

# Conditional update
yq -i 'if .env == "prod" then .replicas = 5 else . end' file.yaml

# Update by selection
yq -i '(.items[] | select(.name == "target")).value = "new"' file.yaml

# Merge
yq -i '. *= {"new": "data"}' file.yaml
```
