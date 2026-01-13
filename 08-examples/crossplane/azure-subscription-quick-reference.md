# Azure Subscription Crossplane - Quick Reference

A comprehensive guide for managing Azure Subscriptions using Crossplane with yq for YAML manipulation.

## Overview

This guide covers working with Azure Subscription XRDs and Compositions for enterprise platform engineering, focusing on subscription governance, cost management, and security.

## Files in This Guide

1. **azure-subscription-xrd.md** - Composite Resource Definition (API schema)
1. **azure-subscription-composition.md** - Implementation with Azure resources
1. **This file** - Quick reference and common patterns

## Common yq Operations

### XRD Modifications

```bash
# Add new parameter field
yq -i '
  .spec.versions[0].schema.openAPIV3Schema.properties.spec.properties.parameters.properties.newField = {
    "type": "string",
    "description": "New field description"
  }
' azure-subscription-xrd.yaml

# Update enum values
yq -i '
  .spec.versions[0].schema.openAPIV3Schema.properties.spec.properties.parameters.properties.environment.enum += ["pre-prod"]
' azure-subscription-xrd.yaml

# Change default value
yq -i '
  .spec.versions[0].schema.openAPIV3Schema.properties.spec.properties.parameters.properties.budget.properties.amount.default = 5000
' azure-subscription-xrd.yaml

# Add printer column
yq -i '
  .spec.versions[0].additionalPrinterColumns += [{
    "name": "Owner",
    "type": "string",
    "jsonPath": ".spec.parameters.rbac.owners[0]"
  }]
' azure-subscription-xrd.yaml
```

### Composition Modifications

```bash
# Update provider config for all resources
yq -i '
  .spec.resources[].base.spec.providerConfigRef.name = "new-provider"
' azure-subscription-composition.yaml

# Add resource to composition
yq -i '
  .spec.resources += [{
    "name": "new-resource",
    "base": {
      "apiVersion": "example.io/v1",
      "kind": "Resource",
      "spec": {}
    }
  }]
' azure-subscription-composition.yaml

# Update specific resource
yq -i '
  (.spec.resources[] | select(.name == "budget").base.spec.forProvider.amount) = 10000
' azure-subscription-composition.yaml

# Add patch to resource
yq -i '
  (.spec.resources[] | select(.name == "subscription").patches) += [{
    "type": "FromCompositeFieldPath",
    "fromFieldPath": "spec.parameters.newField",
    "toFieldPath": "spec.forProvider.newField"
  }]
' azure-subscription-composition.yaml
```

### Claim Modifications

```bash
# Update budget amount
yq -i '.spec.parameters.budget.amount = 15000' claim.yaml

# Add RBAC assignment
yq -i '.spec.parameters.rbac.contributors += ["newuser@example.com"]' claim.yaml

# Change environment
yq -i '.spec.parameters.environment = "production"' claim.yaml

# Add tag
yq -i '.spec.parameters.tags.compliance = "pci-dss"' claim.yaml

# Update policy assignment
yq -i '
  (.spec.parameters.policies.assignments[] | 
   select(.name == "allowed-locations").parameters.listOfAllowedLocations) += ["eastus"]
' claim.yaml
```

## Validation Commands

```bash
# Validate XRD structure
yq '.apiVersion' xrd.yaml | grep -q "apiextensions.crossplane.io" && echo "Valid XRD"

# Validate Composition references XRD
yq '.spec.compositeTypeRef.kind' composition.yaml

# Check all resources have names
yq '.spec.resources[] | select(.name == null)' composition.yaml

# Verify required XRD fields
yq '.spec.versions[0].schema.openAPIV3Schema.properties.spec.properties.parameters.required[]' xrd.yaml

# Check claim matches XRD
kubectl crossplane beta validate claim.yaml --xrd xrd.yaml
```

## Extraction and Reporting

```bash
# List all XRD parameters
yq '.spec.versions[0].schema.openAPIV3Schema.properties.spec.properties.parameters.properties | keys' xrd.yaml

# Get all composition resource names
yq '.spec.resources[].name' composition.yaml

# Extract budget configuration
yq '.spec.parameters.budget' claim.yaml

# List all patches in composition
yq '.spec.resources[] | select(.patches != null) | {name: .name, patches: .patches | length}' composition.yaml

# Get policy assignments
yq '.spec.parameters.policies.assignments[] | {name: .name, policyId: .policyDefinitionId}' claim.yaml

# Extract RBAC configuration
yq '.spec.parameters.rbac' claim.yaml
```

## Environment-Specific Patterns

### Development Environment

```bash
# Create dev composition variant
yq '
  .metadata.name = "azure-subscription-dev" |
  .metadata.labels.environment = "dev" |
  (.spec.resources[] | select(.name == "budget").base.spec.forProvider.amount) = 1000 |
  (.spec.resources[] | select(.metadata.labels."resource-type" == "policy").base.spec.forProvider.enforcementMode) = "DoNotEnforce"
' composition.yaml > composition-dev.yaml

# Create dev claim
yq '
  .metadata.name = "my-dev-subscription" |
  .spec.parameters.environment = "dev" |
  .spec.parameters.budget.amount = 1000 |
  .spec.parameters.networking.allowedRegions = ["westeurope"]
' claim-template.yaml > claim-dev.yaml
```

### Production Environment

```bash
# Create prod composition variant
yq '
  .metadata.name = "azure-subscription-prod" |
  .metadata.labels.environment = "prod" |
  (.spec.resources[] | select(.name == "budget").base.spec.forProvider.amount) = 50000 |
  (.spec.resources[] | select(.name == "log-analytics").base.spec.forProvider.retentionInDays) = 365 |
  (.spec.resources[] | select(.metadata.labels."resource-type" == "policy").base.spec.forProvider.enforcementMode) = "Default"
' composition.yaml > composition-prod.yaml

# Create prod claim
yq '
  .metadata.name = "my-prod-subscription" |
  .spec.parameters.environment = "production" |
  .spec.parameters.budget.amount = 50000 |
  .spec.parameters.networking.allowedRegions = ["westeurope", "northeurope"] |
  .spec.parameters.policies.enabled = true
' claim-template.yaml > claim-prod.yaml
```

## Automation Scripts

### Bulk Update Script

```bash
#!/bin/bash
# bulk-update-subscriptions.sh

CLAIMS_DIR="./claims"
NEW_BUDGET=8000

for claim in "$CLAIMS_DIR"/*.yaml; do
  echo "Updating $claim..."
  yq -i ".spec.parameters.budget.amount = $NEW_BUDGET" "$claim"
done
```

### Compliance Audit Script

```bash
#!/bin/bash
# audit-subscriptions.sh

echo "Subscription Compliance Audit"
echo "=============================="

for claim in ./claims/*.yaml; do
  name=$(yq '.metadata.name' "$claim")
  budget=$(yq '.spec.parameters.budget.amount' "$claim")
  policies=$(yq '.spec.parameters.policies.enabled' "$claim")
  env=$(yq '.spec.parameters.environment' "$claim")
  
  echo ""
  echo "Subscription: $name"
  echo "  Environment: $env"
  echo "  Budget: €$budget"
  echo "  Policies Enabled: $policies"
  
  # Check for required tags
  if ! yq -e '.spec.parameters.tags.environment' "$claim" > /dev/null 2>&1; then
    echo "  ⚠️  Missing environment tag"
  fi
  
  if ! yq -e '.spec.parameters.tags.costCenter' "$claim" > /dev/null 2>&1; then
    echo "  ⚠️  Missing costCenter tag"
  fi
done
```

### Cost Summary Script

```bash
#!/bin/bash
# cost-summary.sh

echo "Azure Subscription Cost Summary"
echo "================================"

total_budget=0
claims=0

for claim in ./claims/*.yaml; do
  name=$(yq '.metadata.name' "$claim")
  budget=$(yq '.spec.parameters.budget.amount' "$claim")
  env=$(yq '.spec.parameters.environment' "$claim")
  
  total_budget=$((total_budget + budget))
  claims=$((claims + 1))
  
  printf "%-30s %-15s €%10d\n" "$name" "$env" "$budget"
done

echo "================================"
printf "%-30s %-15s €%10d\n" "TOTAL ($claims subscriptions)" "" "$total_budget"
```

## GitOps Workflow

### Pre-Commit Hook

```bash
#!/bin/bash
# .git/hooks/pre-commit

echo "Validating Crossplane resources..."

# Validate all XRDs
for xrd in crossplane/xrds/*.yaml; do
  if ! yq '.' "$xrd" > /dev/null 2>&1; then
    echo "ERROR: Invalid YAML in $xrd"
    exit 1
  fi
done

# Validate all Compositions
for comp in crossplane/compositions/*.yaml; do
  if ! yq '.' "$comp" > /dev/null 2>&1; then
    echo "ERROR: Invalid YAML in $comp"
    exit 1
  fi
  
  # Check compositeTypeRef exists
  if ! yq -e '.spec.compositeTypeRef' "$comp" > /dev/null 2>&1; then
    echo "ERROR: Missing compositeTypeRef in $comp"
    exit 1
  fi
done

# Validate all Claims
for claim in crossplane/claims/*.yaml; do
  if ! yq '.' "$claim" > /dev/null 2>&1; then
    echo "ERROR: Invalid YAML in $claim"
    exit 1
  fi
done

echo "✓ All validations passed"
```

### CI/CD Pipeline (GitHub Actions)

```yaml
name: Validate Crossplane Resources

on: [push, pull_request]

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    - name: Install yq
      run: |
        wget https://github.com/mikefarah/yq/releases/latest/download/yq_linux_amd64 -O /usr/local/bin/yq
        chmod +x /usr/local/bin/yq
    
    - name: Validate XRDs
      run: |
        for xrd in crossplane/xrds/*.yaml; do
          yq '.' "$xrd" > /dev/null
          echo "✓ $xrd"
        done
    
    - name: Validate Compositions
      run: |
        for comp in crossplane/compositions/*.yaml; do
          yq '.' "$comp" > /dev/null
          kind=$(yq '.kind' "$comp")
          if [ "$kind" != "Composition" ]; then
            echo "ERROR: Invalid kind in $comp"
            exit 1
          fi
          echo "✓ $comp"
        done
    
    - name: Validate Claims
      run: |
        for claim in crossplane/claims/*.yaml; do
          yq '.' "$claim" > /dev/null
          echo "✓ $claim"
        done
```

## Troubleshooting

### Common Issues

**Missing Required Fields in Claim:**

```bash
# Check what's required
yq '.spec.versions[0].schema.openAPIV3Schema.properties.spec.properties.parameters.required[]' xrd.yaml

# Verify claim has all required fields
for field in $(yq '.spec.versions[0].schema.openAPIV3Schema.properties.spec.properties.parameters.required[]' xrd.yaml); do
  if ! yq -e ".spec.parameters.$field" claim.yaml > /dev/null 2>&1; then
    echo "Missing required field: $field"
  fi
done
```

**Composition Not Matching Claim:**

```bash
# Check composition selector
yq '.spec.compositionSelector.matchLabels' claim.yaml

# Check composition labels
yq '.metadata.labels' composition.yaml
```

**Invalid Budget Configuration:**

```bash
# Validate budget amount is within limits
amount=$(yq '.spec.parameters.budget.amount' claim.yaml)
min=$(yq '.spec.versions[0].schema.openAPIV3Schema.properties.spec.properties.parameters.properties.budget.properties.amount.minimum' xrd.yaml)

if [ "$amount" -lt "$min" ]; then
  echo "Budget amount $amount is below minimum $min"
fi
```

## Best Practices Summary

1. **Version Control**: Always version your XRDs and Compositions
1. **Validation**: Validate YAML before committing
1. **Defaults**: Provide sensible defaults in XRD
1. **Required Fields**: Keep required fields to minimum
1. **Documentation**: Document parameters with descriptions
1. **Tags**: Enforce consistent tagging through policies
1. **Budgets**: Always set budget alerts
1. **RBAC**: Follow least privilege principle
1. **Policies**: Use DoNotEnforce in dev, Default in prod
1. **Testing**: Test with sample claims before applying

## Related Documentation

- [XRD Definition](./azure-subscription-xrd.md) - Complete XRD with all parameters
- [Composition](./azure-subscription-composition.md) - Full composition with all resources
- [AKS Composition](./azure-aks-composition.md) - Kubernetes cluster example
- [yq Documentation](https://mikefarah.gitbook.io/yq/) - Official yq docs

## Quick Tips

```bash
# Pretty print any YAML
yq -P '.' file.yaml

# Convert to JSON for processing
yq -o json '.' file.yaml

# Validate syntax
yq '.' file.yaml > /dev/null && echo "Valid"

# Format all YAML in directory
find . -name "*.yaml" -exec yq -i -P '.' {} \;

# Search for specific value
yq '.. | select(. == "production")' file.yaml

# Count resources in composition
yq '.spec.resources | length' composition.yaml

# Extract emails from RBAC
yq '.spec.parameters.rbac | .. | select(test("@"))' claim.yaml
```

This quick reference provides the most common patterns for working with Azure Subscription Crossplane resources using yq.
