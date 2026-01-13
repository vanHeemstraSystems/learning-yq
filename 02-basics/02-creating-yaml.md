# Creating YAML with yq

## Overview

This guide covers how to create YAML files from scratch using yq, a crucial skill for generating Kubernetes manifests, Crossplane resources, and configuration files programmatically.

## Basic Document Creation

### Creating a New Empty Document

```bash
# Create empty YAML document
yq -n '.' > empty.yaml

# Create document with null
yq -n 'null' > null.yaml
```

### Creating Simple Key-Value Pairs

```bash
# Single field
yq -n '.name = "my-app"' > config.yaml

# Multiple fields with pipe operator
yq -n '
  .name = "my-app" |
  .version = "1.0.0" |
  .enabled = true
' > config.yaml

# Output:
# name: my-app
# version: 1.0.0
# enabled: true
```

### Creating Nested Structures

```bash
# Nested object
yq -n '
  .metadata.name = "web-service" |
  .metadata.namespace = "production" |
  .spec.port = 8080
' > service.yaml

# Output:
# metadata:
#   name: web-service
#   namespace: production
# spec:
#   port: 8080
```

## Working with Different Data Types

### Strings

```bash
# Plain string
yq -n '.message = "Hello World"'

# String with quotes
yq -n '.message = "He said \"Hello\""'

# Multiline string (literal)
yq -n '.description = "Line 1\nLine 2\nLine 3"'

# String from variable
MESSAGE="Production deployment"
yq -n ".message = \"$MESSAGE\""
```

### Numbers

```bash
# Integer
yq -n '.port = 8080'

# Float
yq -n '.version = 1.5'

# Negative number
yq -n '.offset = -10'

# Scientific notation
yq -n '.value = 1e6'
```

### Booleans

```bash
# Boolean values
yq -n '.enabled = true'
yq -n '.disabled = false'

# Boolean from variable
ENABLED=true
yq -n ".enabled = $ENABLED"
```

### Null Values

```bash
# Explicit null
yq -n '.value = null'

# Null from missing field
yq -n '.existing = "value" | .missing = null'
```

## Creating Arrays

### Simple Arrays

```bash
# Array of strings
yq -n '.colors = ["red", "green", "blue"]'

# Array of numbers
yq -n '.ports = [80, 443, 8080]'

# Empty array
yq -n '.items = []'
```

### Adding Items to Arrays

```bash
# Create array and add items
yq -n '
  .items = [] |
  .items += "first" |
  .items += "second" |
  .items += "third"
'

# Create array with multiple items at once
yq -n '
  .items = [] |
  .items += ["first", "second", "third"]
'
```

### Arrays of Objects

```bash
# Array of objects
yq -n '
  .containers = [
    {"name": "app", "image": "nginx:1.21"},
    {"name": "sidecar", "image": "busybox:latest"}
  ]
'

# Adding objects to array incrementally
yq -n '
  .containers = [] |
  .containers += {"name": "app", "image": "nginx:1.21"} |
  .containers += {"name": "sidecar", "image": "busybox:latest"}
'
```

## Kubernetes Examples

### Creating a ConfigMap

```bash
yq -n '
  .apiVersion = "v1" |
  .kind = "ConfigMap" |
  .metadata.name = "app-config" |
  .metadata.namespace = "default" |
  .data.DATABASE_URL = "postgres://localhost:5432/mydb" |
  .data.LOG_LEVEL = "info" |
  .data.MAX_CONNECTIONS = "100"
' > configmap.yaml
```

### Creating a Secret

```bash
# Note: Values should be base64 encoded
USERNAME=$(echo -n "admin" | base64)
PASSWORD=$(echo -n "secret123" | base64)

yq -n "
  .apiVersion = \"v1\" |
  .kind = \"Secret\" |
  .metadata.name = \"db-credentials\" |
  .metadata.namespace = \"default\" |
  .type = \"Opaque\" |
  .data.username = \"$USERNAME\" |
  .data.password = \"$PASSWORD\"
" > secret.yaml
```

### Creating a Deployment

```bash
yq -n '
  .apiVersion = "apps/v1" |
  .kind = "Deployment" |
  .metadata.name = "nginx-deployment" |
  .metadata.namespace = "default" |
  .metadata.labels.app = "nginx" |
  .spec.replicas = 3 |
  .spec.selector.matchLabels.app = "nginx" |
  .spec.template.metadata.labels.app = "nginx" |
  .spec.template.spec.containers = [{
    "name": "nginx",
    "image": "nginx:1.21",
    "ports": [{"containerPort": 80}]
  }]
' > deployment.yaml
```

### Creating a Service

```bash
yq -n '
  .apiVersion = "v1" |
  .kind = "Service" |
  .metadata.name = "nginx-service" |
  .metadata.namespace = "default" |
  .spec.type = "ClusterIP" |
  .spec.selector.app = "nginx" |
  .spec.ports = [{
    "protocol": "TCP",
    "port": 80,
    "targetPort": 80
  }]
' > service.yaml
```

### Creating a Namespace

```bash
yq -n '
  .apiVersion = "v1" |
  .kind = "Namespace" |
  .metadata.name = "my-namespace" |
  .metadata.labels.environment = "production" |
  .metadata.labels.team = "platform"
' > namespace.yaml
```

## Crossplane Examples

### Creating a Simple XRD

```bash
yq -n '
  .apiVersion = "apiextensions.crossplane.io/v1" |
  .kind = "CompositeResourceDefinition" |
  .metadata.name = "xdatabases.example.com" |
  .spec.group = "example.com" |
  .spec.names.kind = "XDatabase" |
  .spec.names.plural = "xdatabases" |
  .spec.versions = [{
    "name": "v1alpha1",
    "served": true,
    "referenceable": true
  }]
' > database-xrd.yaml
```

### Creating a Basic Composition

```bash
yq -n '
  .apiVersion = "apiextensions.crossplane.io/v1" |
  .kind = "Composition" |
  .metadata.name = "azure-database" |
  .metadata.labels.provider = "azure" |
  .spec.compositeTypeRef.apiVersion = "example.com/v1alpha1" |
  .spec.compositeTypeRef.kind = "XDatabase" |
  .spec.resources = []
' > composition.yaml
```

### Creating a Claim

```bash
yq -n '
  .apiVersion = "example.com/v1alpha1" |
  .kind = "Database" |
  .metadata.name = "my-database" |
  .metadata.namespace = "default" |
  .spec.parameters.size = "small" |
  .spec.parameters.engine = "postgresql" |
  .spec.parameters.version = "14"
' > database-claim.yaml
```

## Advanced Creation Patterns

### Using Variables

```bash
# Define variables
APP_NAME="my-application"
NAMESPACE="production"
REPLICAS=5
IMAGE="nginx:1.21"

# Create deployment with variables
yq -n "
  .apiVersion = \"apps/v1\" |
  .kind = \"Deployment\" |
  .metadata.name = \"$APP_NAME\" |
  .metadata.namespace = \"$NAMESPACE\" |
  .spec.replicas = $REPLICAS |
  .spec.template.spec.containers = [{
    \"name\": \"$APP_NAME\",
    \"image\": \"$IMAGE\"
  }]
" > deployment.yaml
```

### Conditional Creation

```bash
# Create resource with conditional fields
ENABLE_AUTOSCALING=true

yq -n "
  .apiVersion = \"apps/v1\" |
  .kind = \"Deployment\" |
  .metadata.name = \"my-app\" |
  .spec.replicas = $([ "$ENABLE_AUTOSCALING" = true ] && echo 1 || echo 3)
" > deployment.yaml
```

### Creating from Templates

```bash
# Define template function
create_deployment() {
  local name=$1
  local image=$2
  local replicas=$3
  
  yq -n "
    .apiVersion = \"apps/v1\" |
    .kind = \"Deployment\" |
    .metadata.name = \"$name\" |
    .spec.replicas = $replicas |
    .spec.template.spec.containers = [{
      \"name\": \"$name\",
      \"image\": \"$image\"
    }]
  "
}

# Use template
create_deployment "nginx" "nginx:1.21" 3 > nginx-deployment.yaml
create_deployment "redis" "redis:7" 1 > redis-deployment.yaml
```

### Merging During Creation

```bash
# Create base configuration
yq -n '
  .apiVersion = "v1" |
  .kind = "ConfigMap" |
  .metadata.name = "base-config" |
  .data.ENVIRONMENT = "production"
' > base.yaml

# Create additional configuration and merge
yq -n '
  .data.DATABASE_URL = "postgres://db:5432" |
  .data.CACHE_URL = "redis://cache:6379"
' > additional.yaml

# Merge them
yq eval-all 'select(fileIndex == 0) * select(fileIndex == 1)' base.yaml additional.yaml > merged.yaml
```

## Creating from JSON

### Convert JSON to YAML

```bash
# From JSON file
yq -P '.' input.json > output.yaml

# From JSON string
echo '{"name": "app", "version": "1.0"}' | yq -P '.'

# From environment variable
JSON_DATA='{"app": "nginx", "replicas": 3}'
echo "$JSON_DATA" | yq -P '.' > config.yaml
```

### Creating YAML from JSON Structure

```bash
# Create deployment from JSON structure
JSON_CONFIG='{
  "name": "my-app",
  "image": "nginx:1.21",
  "replicas": 3,
  "port": 80
}'

yq -n --from-json "$JSON_CONFIG" | yq '
  .apiVersion = "apps/v1" |
  .kind = "Deployment" |
  .metadata.name = .name |
  .spec.replicas = .replicas |
  .spec.template.spec.containers = [{
    "name": .name,
    "image": .image,
    "ports": [{"containerPort": .port}]
  }] |
  del(.name, .image, .port)
' > deployment.yaml
```

## Practical Automation Scripts

### Generate Multiple Resources

```bash
#!/bin/bash
# create-namespaces.sh

NAMESPACES=("dev" "staging" "production")

for ns in "${NAMESPACES[@]}"; do
  yq -n "
    .apiVersion = \"v1\" |
    .kind = \"Namespace\" |
    .metadata.name = \"$ns\" |
    .metadata.labels.environment = \"$ns\"
  " > "namespace-$ns.yaml"
  echo "Created namespace-$ns.yaml"
done
```

### Generate from CSV

```bash
#!/bin/bash
# create-configmaps-from-csv.sh

# CSV format: name,key1,value1,key2,value2
# Example: app1,DB_HOST,localhost,DB_PORT,5432

while IFS=',' read -r name key1 val1 key2 val2; do
  yq -n "
    .apiVersion = \"v1\" |
    .kind = \"ConfigMap\" |
    .metadata.name = \"$name-config\" |
    .data.$key1 = \"$val1\" |
    .data.$key2 = \"$val2\"
  " > "configmap-$name.yaml"
done < config-data.csv
```

### Generate Environment-Specific Configs

```bash
#!/bin/bash
# create-env-configs.sh

create_config() {
  local env=$1
  local replicas=$2
  local cpu=$3
  local memory=$4
  
  yq -n "
    .apiVersion = \"v1\" |
    .kind = \"ConfigMap\" |
    .metadata.name = \"app-config-$env\" |
    .metadata.labels.environment = \"$env\" |
    .data.ENVIRONMENT = \"$env\" |
    .data.REPLICAS = \"$replicas\" |
    .data.CPU_LIMIT = \"$cpu\" |
    .data.MEMORY_LIMIT = \"$memory\"
  " > "config-$env.yaml"
}

# Create configs for each environment
create_config "dev" 1 "100m" "128Mi"
create_config "staging" 2 "200m" "256Mi"
create_config "production" 5 "500m" "512Mi"
```

## Validation After Creation

```bash
# Create and validate in one command
yq -n '
  .apiVersion = "v1" |
  .kind = "ConfigMap" |
  .metadata.name = "my-config"
' > config.yaml && yq '.' config.yaml && echo "✓ Valid YAML"

# Create with error checking
if yq -n '.name = "app"' > config.yaml 2>&1; then
  echo "Created config.yaml successfully"
else
  echo "Failed to create config.yaml"
  exit 1
fi
```

## Best Practices

### 1. Use Heredoc for Complex Documents

```bash
cat > deployment.yaml << 'EOF'
$(yq -n '
  .apiVersion = "apps/v1" |
  .kind = "Deployment" |
  .metadata.name = "complex-app" |
  .spec.replicas = 3
')
EOF
```

### 2. Create with Proper Formatting

```bash
# Always use -P for pretty printing
yq -n -P '.name = "app"' > config.yaml

# Or format after creation
yq -n '.name = "app"' | yq -P '.' > config.yaml
```

### 3. Validate Data Types

```bash
# Ensure numbers are not quoted
yq -n '.port = 8080' > config.yaml  # Correct
yq -n '.port = "8080"' > config.yaml  # Wrong (string instead of number)

# Verify
yq '.port | type' config.yaml  # Should show !!int
```

### 4. Use Functions for Reusability

```bash
# Create reusable function
create_label() {
  local key=$1
  local value=$2
  echo ".metadata.labels.$key = \"$value\""
}

# Use in creation
yq -n "
  .apiVersion = \"v1\" |
  .kind = \"ConfigMap\" |
  .metadata.name = \"my-config\" |
  $(create_label "app" "myapp") |
  $(create_label "env" "prod")
" > config.yaml
```

### 5. Handle Special Characters

```bash
# Escape quotes in strings
yq -n '.message = "He said \"Hello\""'

# Use single quotes for literal strings
yq -n ".message = 'It'\''s working'"

# Use variables for complex strings
MESSAGE="Path: C:\Users\Admin"
yq -n ".path = \"$MESSAGE\""
```

## Common Patterns

### Creating Labels and Annotations

```bash
yq -n '
  .metadata.labels = {
    "app": "myapp",
    "version": "1.0",
    "environment": "production"
  } |
  .metadata.annotations = {
    "description": "Main application",
    "owner": "platform-team"
  }
'
```

### Creating Resource Limits

```bash
yq -n '
  .resources.requests.cpu = "100m" |
  .resources.requests.memory = "128Mi" |
  .resources.limits.cpu = "200m" |
  .resources.limits.memory = "256Mi"
'
```

### Creating Environment Variables

```bash
yq -n '
  .env = [
    {"name": "DATABASE_URL", "value": "postgres://db:5432"},
    {"name": "CACHE_URL", "value": "redis://cache:6379"},
    {"name": "LOG_LEVEL", "value": "info"}
  ]
'
```

## Troubleshooting

### Issue: Invalid YAML Generated

```bash
# Always validate after creation
yq -n '.name = "app"' > config.yaml
yq '.' config.yaml > /dev/null && echo "Valid" || echo "Invalid"
```

### Issue: Incorrect Data Types

```bash
# Check type
yq '.port | type' config.yaml

# Convert if needed
yq -n '.port = "8080"' | yq '.port |= tonumber'
```

### Issue: Missing Quotes

```bash
# Values with special characters need quotes
yq -n '.path = "/usr/local/bin"'  # Correct
yq -n .path = /usr/local/bin  # Wrong (syntax error)
```

## Next Steps

- [Editing YAML](./03-editing-yaml.md)
- [Working with Data Types](./04-data-types.md)
- [Advanced Operations](../03-advanced/)

## Quick Reference

```bash
# Basic creation
yq -n '.key = "value"' > file.yaml

# Multiple fields
yq -n '.key1 = "value1" | .key2 = "value2"' > file.yaml

# Nested structure
yq -n '.parent.child = "value"' > file.yaml

# Array
yq -n '.items = ["a", "b", "c"]' > file.yaml

# Object in array
yq -n '.items = [{"name": "item1"}]' > file.yaml

# Using variables
VAR="value"
yq -n ".key = \"$VAR\"" > file.yaml

# Pretty print
yq -n -P '.key = "value"' > file.yaml

# Convert JSON to YAML
yq -P '.' input.json > output.yaml
```
