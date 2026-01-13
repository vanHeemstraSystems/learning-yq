# Working with Data Types in yq

## Overview

Understanding and manipulating different data types is crucial for effective YAML processing with yq. This guide covers all YAML data types and their conversions.

## YAML Data Types

### Strings

Strings are the most common data type in YAML.

**Plain strings:**

```bash
# Create plain string
yq -n '.message = "Hello World"'

# Read string
yq '.message' config.yaml

# Check if value is string
yq '.message | type' config.yaml
# Output: !!str
```

**String with special characters:**

```bash
# Quotes in strings
yq -n '.message = "He said \"Hello\""'

# Single quotes (literal)
yq -n ".message = 'It'\''s working'"

# Multiline strings
yq -n '.description = "Line 1\nLine 2\nLine 3"'
```

**String operations:**

```bash
# Concatenation
yq -n '.full_name = "John" + " " + "Doe"'

# Uppercase
yq '.name | upcase' config.yaml

# Lowercase
yq '.name | downcase' config.yaml

# Length
yq '.message | length' config.yaml

# Substring
yq '.message | .[0:5]' config.yaml

# Replace
yq '.message | sub("old", "new")' config.yaml

# Split
yq '.path | split("/")' config.yaml

# Join
yq '.words | join("-")' config.yaml

# Test/match
yq '.email | test("@example.com")' config.yaml

# Contains
yq '.message | contains("error")' config.yaml
```

### Numbers

YAML supports both integers and floating-point numbers.

**Integers:**

```bash
# Create integer
yq -n '.count = 42'

# Check type
yq '.count | type' config.yaml
# Output: !!int

# Arithmetic operations
yq '.count + 10' config.yaml
yq '.count - 5' config.yaml
yq '.count * 2' config.yaml
yq '.count / 3' config.yaml
yq '.count % 5' config.yaml  # Modulo
```

**Floating-point numbers:**

```bash
# Create float
yq -n '.version = 1.5'

# Check type
yq '.version | type' config.yaml
# Output: !!float

# Arithmetic with floats
yq '.version + 0.5' config.yaml

# Round
yq '.version | floor' config.yaml  # Round down
yq '.version | ceil' config.yaml   # Round up
yq '.version | round' config.yaml  # Round to nearest
```

**Number conversions:**

```bash
# String to number
yq '.port | tonumber' config.yaml

# Number to string
yq '.count | tostring' config.yaml

# Ensure integer
yq '.value | floor | tonumber' config.yaml
```

### Booleans

Boolean values are `true` or `false`.

**Creating booleans:**

```bash
# Boolean true
yq -n '.enabled = true'

# Boolean false
yq -n '.disabled = false'

# From string
yq -n '.enabled = "true" | .enabled |= (. == "true")'
```

**Boolean operations:**

```bash
# Logical AND
yq '.enabled and .ready' config.yaml

# Logical OR
yq '.enabled or .ready' config.yaml

# Logical NOT
yq '.enabled | not' config.yaml

# Comparison
yq '.count > 5' config.yaml
yq '.count >= 5' config.yaml
yq '.count < 10' config.yaml
yq '.count <= 10' config.yaml
yq '.count == 5' config.yaml
yq '.count != 5' config.yaml
```

**Boolean conversions:**

```bash
# Any value to boolean
yq '.value | . != null' config.yaml

# String to boolean
yq '.enabled = "true" | .enabled |= (. == "true")' config.yaml

# Number to boolean (0 is false, others true)
yq '.count | . != 0' config.yaml
```

### Null

Null represents absence of a value.

**Working with null:**

```bash
# Create null
yq -n '.optional = null'

# Check for null
yq '.field == null' config.yaml

# Check if field exists
yq 'has("field")' config.yaml

# Default value if null
yq '.field // "default"' config.yaml

# Remove null values
yq 'del(.[] | select(. == null))' config.yaml
```

### Arrays

Arrays (sequences) are ordered collections.

**Creating arrays:**

```bash
# Array of strings
yq -n '.colors = ["red", "green", "blue"]'

# Array of numbers
yq -n '.ports = [80, 443, 8080]'

# Mixed array
yq -n '.mixed = ["text", 42, true, null]'

# Empty array
yq -n '.items = []'

# Array of objects
yq -n '.users = [
  {"name": "Alice", "age": 30},
  {"name": "Bob", "age": 25}
]'
```

**Array operations:**

```bash
# Length
yq '.items | length' config.yaml

# Access by index
yq '.items[0]' config.yaml
yq '.items[-1]' config.yaml  # Last item

# Slice
yq '.items[1:3]' config.yaml  # Items 1 and 2
yq '.items[:2]' config.yaml   # First 2 items
yq '.items[2:]' config.yaml   # From index 2 to end

# Add to array
yq '.items += "new-item"' config.yaml
yq '.items = .items + ["item1", "item2"]' config.yaml

# Filter array
yq '.items[] | select(. > 5)' config.yaml

# Map array
yq '.items | map(. * 2)' config.yaml

# Sort
yq '.items | sort' config.yaml
yq '.items | sort_by(.name)' config.yaml

# Unique
yq '.items | unique' config.yaml

# Reverse
yq '.items | reverse' config.yaml

# Contains element
yq '.items | contains(["target"])' config.yaml

# Find index
yq '.items | index("target")' config.yaml

# Flatten
yq '.nested_arrays | flatten' config.yaml

# Group by
yq '.items | group_by(.category)' config.yaml
```

### Objects

Objects (mappings) are key-value pairs.

**Creating objects:**

```bash
# Simple object
yq -n '.user = {"name": "Alice", "age": 30}'

# Nested object
yq -n '.config = {
  "database": {
    "host": "localhost",
    "port": 5432
  }
}'

# Empty object
yq -n '.metadata = {}'
```

**Object operations:**

```bash
# Get keys
yq '.user | keys' config.yaml

# Get values
yq '.user | to_entries | .[] | .value' config.yaml

# Check if key exists
yq '.user | has("name")' config.yaml

# Add field
yq '.user.email = "alice@example.com"' config.yaml

# Remove field
yq 'del(.user.age)' config.yaml

# Merge objects
yq '.config *= {"new": "value"}' config.yaml

# Convert to array of key-value pairs
yq '.user | to_entries' config.yaml
# Output:
# - key: name
#   value: Alice
# - key: age
#   value: 30

# Convert back from entries
yq '.entries | from_entries' config.yaml

# Select by value
yq '.user | to_entries | .[] | select(.value > 25)' config.yaml
```

## Type Checking

### Check Type

```bash
# Get type
yq '.field | type' config.yaml

# Check specific type
yq '.field | type == "!!str"' config.yaml
yq '.field | type == "!!int"' config.yaml
yq '.field | type == "!!float"' config.yaml
yq '.field | type == "!!bool"' config.yaml
yq '.field | type == "!!null"' config.yaml
```

### Type Assertions

```bash
# Assert string
if [ "$(yq '.name | type' config.yaml)" == "!!str" ]; then
  echo "name is a string"
fi

# Assert number
if yq '.port | type' config.yaml | grep -q "int\|float"; then
  echo "port is a number"
fi

# Assert boolean
if [ "$(yq '.enabled | type' config.yaml)" == "!!bool" ]; then
  echo "enabled is a boolean"
fi
```

## Type Conversions

### To String

```bash
# Number to string
yq '.count | tostring' config.yaml

# Boolean to string
yq '.enabled | tostring' config.yaml

# Array to string (JSON)
yq '.items | tojson' config.yaml

# Object to string (JSON)
yq '.config | tojson' config.yaml
```

### To Number

```bash
# String to number
yq '.port | tonumber' config.yaml

# String to integer
yq '"42" | tonumber' config.yaml

# String to float
yq '"3.14" | tonumber' config.yaml

# Boolean to number (true=1, false=0)
yq '.enabled | if . then 1 else 0 end' config.yaml
```

### To Boolean

```bash
# String to boolean
yq '.enabled | . == "true"' config.yaml

# Number to boolean
yq '.count | . != 0' config.yaml

# Null to boolean
yq '.field | . != null' config.yaml
```

### To Array

```bash
# String to array (split)
yq '.path | split("/")' config.yaml

# Single value to array
yq '.item | [.]' config.yaml

# Object to array of values
yq '.config | [.[]]' config.yaml
```

### To Object

```bash
# Array of entries to object
yq '[{"key": "name", "value": "Alice"}] | from_entries' config.yaml

# Create object from values
yq '{
  "name": .firstName + " " + .lastName,
  "age": .age
}' config.yaml
```

## Kubernetes-Specific Type Handling

### Resource Quantities

```bash
# CPU values (strings that look like numbers)
yq -n '.resources.limits.cpu = "500m"'
yq -n '.resources.limits.cpu = "2"'

# Memory values
yq -n '.resources.limits.memory = "512Mi"'
yq -n '.resources.limits.memory = "2Gi"'

# Storage
yq -n '.storage = "10Gi"'

# Parse quantity
yq '.resources.limits.memory | sub("Mi", "") | tonumber' config.yaml
```

### Port Numbers

```bash
# Ensure ports are integers, not strings
yq -n '.port = 8080'  # Correct
yq -n '.port = "8080" | .port |= tonumber'  # Convert string to int

# Validate port is integer
if yq '.port | type' config.yaml | grep -q "int"; then
  echo "Port is correctly an integer"
fi
```

### Environment Variables

```bash
# Env vars are typically strings
yq -n '.env = [
  {"name": "PORT", "value": "8080"},
  {"name": "DEBUG", "value": "true"}
]'

# Convert to proper types if needed
yq '.env[] | select(.name == "PORT") | .value | tonumber' config.yaml
```

## Crossplane-Specific Type Handling

### Schema Types

```bash
# Integer parameters
yq -n '.spec.parameters.nodeCount = 3'

# String parameters
yq -n '.spec.parameters.location = "westeurope"'

# Boolean parameters
yq -n '.spec.parameters.enabled = true'

# Array parameters
yq -n '.spec.parameters.allowedRegions = ["westeurope", "northeurope"]'

# Ensure correct type
yq -i '.spec.parameters.nodeCount |= tonumber' claim.yaml
```

### Patch Transforms

```bash
# Type conversion in patches
yq -n '.patches = [{
  "type": "FromCompositeFieldPath",
  "fromFieldPath": "spec.parameters.count",
  "toFieldPath": "spec.forProvider.count",
  "transforms": [{
    "type": "convert",
    "convert": {
      "toType": "int"
    }
  }]
}]'
```

## Common Type Issues and Solutions

### Issue: String Instead of Number

```yaml
# Wrong
port: "8080"

# Correct
port: 8080
```

**Fix:**

```bash
yq -i '.port |= tonumber' config.yaml
```

### Issue: Number Instead of String

```yaml
# Wrong (for version)
version: 1.21

# Correct
version: "1.21"
```

**Fix:**

```bash
yq -i '.version |= tostring' config.yaml
```

### Issue: Boolean as String

```yaml
# Wrong
enabled: "true"

# Correct
enabled: true
```

**Fix:**

```bash
yq -i '.enabled |= (. == "true")' config.yaml
```

### Issue: Quoted Numbers in YAML

```yaml
# If YAML has quoted numbers
replicas: "3"
```

**Fix:**

```bash
yq -i '.replicas |= tonumber' config.yaml
```

## Validation Scripts

### Validate Types

```bash
#!/bin/bash
# validate-types.sh

FILE="$1"

echo "Validating types in $FILE..."

# Check port is integer
if ! yq '.spec.port | type' "$FILE" | grep -q "int"; then
  echo "ERROR: port must be integer"
  exit 1
fi

# Check enabled is boolean
if ! yq '.spec.enabled | type' "$FILE" | grep -q "bool"; then
  echo "ERROR: enabled must be boolean"
  exit 1
fi

# Check replicas is integer
if ! yq '.spec.replicas | type' "$FILE" | grep -q "int"; then
  echo "ERROR: replicas must be integer"
  exit 1
fi

echo "✓ All types valid"
```

### Convert Types

```bash
#!/bin/bash
# fix-types.sh

FILE="$1"

echo "Fixing types in $FILE..."

# Convert string numbers to integers
yq -i '
  (.spec.port, .spec.replicas) |= tonumber
' "$FILE"

# Convert string booleans to booleans
yq -i '
  .spec.enabled |= (. == "true" or . == true)
' "$FILE"

# Ensure version is string
yq -i '
  .spec.version |= tostring
' "$FILE"

echo "✓ Types fixed"
```

## Best Practices

### 1. Use Correct Types from Start

```bash
# Good
yq -n '
  .port = 8080 |
  .enabled = true |
  .version = "1.0"
'

# Bad
yq -n '
  .port = "8080" |
  .enabled = "true" |
  .version = 1.0
'
```

### 2. Validate Types After Conversion

```bash
# Convert and validate
yq -i '.port |= tonumber' config.yaml

if ! yq '.port | type' config.yaml | grep -q "int"; then
  echo "Conversion failed"
  exit 1
fi
```

### 3. Handle Type Errors Gracefully

```bash
# Try to convert, use default if fails
PORT=$(yq '.port // 8080' config.yaml)
PORT=$(echo "$PORT" | grep -E '^[0-9]+$' && echo "$PORT" || echo "8080")
```

### 4. Document Expected Types

```yaml
# Good: With comments
port: 8080          # integer
enabled: true       # boolean
version: "1.21"     # string
replicas: 3         # integer
```

## Quick Type Reference

```bash
# Check type
yq '.field | type' file.yaml

# Convert to string
yq '.field | tostring' file.yaml

# Convert to number
yq '.field | tonumber' file.yaml

# Convert to boolean
yq '.field | . == "true"' file.yaml

# Array length
yq '.array | length' file.yaml

# Object keys
yq '.object | keys' file.yaml

# Check if exists
yq 'has("field")' file.yaml

# Check if null
yq '.field == null' file.yaml

# Default if null
yq '.field // "default"' file.yaml
```

## Next Steps

- [Array Operations](../03-advanced/01-array-operations.md)
- [Conditional Logic](../03-advanced/02-conditional-logic.md)
- [Operators Reference](../03-advanced/04-operators.md)

## Common Type Patterns

```yaml
# Kubernetes types
apiVersion: v1                    # string
kind: Deployment                  # string
replicas: 3                       # integer
port: 8080                        # integer
enabled: true                     # boolean
resources:
  limits:
    cpu: "500m"                   # string (resource quantity)
    memory: "512Mi"               # string (resource quantity)
labels:                           # object
  app: nginx                      # string
env:                              # array of objects
  - name: PORT                    # string
    value: "8080"                 # string
```
