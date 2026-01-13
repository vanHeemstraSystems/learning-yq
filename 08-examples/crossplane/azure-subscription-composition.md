# Example: Azure Subscription Composition

This Composition implements the Azure Subscription XRD, creating and configuring Azure subscriptions with governance, policies, budgets, and RBAC through Crossplane.

## Composition File

**File: azure-subscription-composition.yaml**

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: azure-subscription-enterprise
  labels:
    provider: azure
    tier: enterprise
    crossplane.io/xrd: xazuresubscriptions.platform.example.com
  annotations:
    crossplane.io/description: "Enterprise Azure subscription with governance and compliance"
    crossplane.io/version: "1.0.0"
spec:
  writeConnectionSecretsToNamespace: crossplane-system
  
  compositeTypeRef:
    apiVersion: platform.example.com/v1alpha1
    kind: XAzureSubscription
  
  resources:
  
  # Azure Subscription
  - name: subscription
    base:
      apiVersion: subscription.azure.upbound.io/v1beta1
      kind: Subscription
      metadata:
        labels:
          resource-type: subscription
      spec:
        forProvider:
          subscriptionName: placeholder
          billingScope: placeholder
          workload: Production
        providerConfigRef:
          name: azure-provider-prod
    
    patches:
    # Basic subscription configuration
    - type: FromCompositeFieldPath
      fromFieldPath: spec.parameters.subscriptionName
      toFieldPath: spec.forProvider.subscriptionName
    
    - type: FromCompositeFieldPath
      fromFieldPath: spec.parameters.billingAccount
      toFieldPath: spec.forProvider.billingScope
      transforms:
      - type: string
        string:
          fmt: "%s/billingProfiles/default/invoiceSections/default"
    
    - type: FromCompositeFieldPath
      fromFieldPath: spec.parameters.environment
      toFieldPath: spec.forProvider.workload
      transforms:
      - type: map
        map:
          dev: DevTest
          test: DevTest
          staging: Production
          production: Production
    
    # Provider config
    - type: FromCompositeFieldPath
      fromFieldPath: spec.resourceConfig.providerConfigName
      toFieldPath: spec.providerConfigRef.name
    
    # Tags
    - type: FromCompositeFieldPath
      fromFieldPath: spec.parameters.tags
      toFieldPath: spec.forProvider.tags
      policy:
        mergeOptions:
          keepMapValues: true
    
    - type: FromCompositeFieldPath
      fromFieldPath: spec.parameters.environment
      toFieldPath: spec.forProvider.tags.environment
    
    - type: FromCompositeFieldPath
      fromFieldPath: spec.parameters.costCenter
      toFieldPath: spec.forProvider.tags.costCenter
    
    # Status patches
    - type: ToCompositeFieldPath
      fromFieldPath: status.atProvider.subscriptionId
      toFieldPath: status.subscriptionId
    
    - type: ToCompositeFieldPath
      fromFieldPath: status.atProvider.tenantId
      toFieldPath: status.tenantId
    
    - type: ToCompositeFieldPath
      fromFieldPath: status.atProvider.state
      toFieldPath: status.state
    
    # Connection details
    connectionDetails:
    - name: subscriptionId
      fromFieldPath: status.atProvider.subscriptionId
    - name: tenantId
      fromFieldPath: status.atProvider.tenantId
    
    # Readiness checks
    readinessChecks:
    - type: MatchString
      fieldPath: status.atProvider.state
      matchString: Enabled
  
  # Management Group Assignment
  - name: management-group-assignment
    base:
      apiVersion: management.azure.upbound.io/v1beta1
      kind: ManagementGroupSubscriptionAssociation
      metadata:
        labels:
          resource-type: management
      spec:
        forProvider:
          managementGroupId: placeholder
          subscriptionIdSelector:
            matchControllerRef: true
        providerConfigRef:
          name: azure-provider-prod
    
    patches:
    - type: FromCompositeFieldPath
      fromFieldPath: spec.parameters.managementGroup
      toFieldPath: spec.forProvider.managementGroupId
      transforms:
      - type: string
        string:
          fmt: "/providers/Microsoft.Management/managementGroups/%s"
    
    - type: FromCompositeFieldPath
      fromFieldPath: spec.resourceConfig.providerConfigName
      toFieldPath: spec.providerConfigRef.name
    
    - type: ToCompositeFieldPath
      fromFieldPath: status.atProvider.id
      toFieldPath: status.managementGroupId
  
  # Budget
  - name: budget
    base:
      apiVersion: consumption.azure.upbound.io/v1beta1
      kind: Budget
      metadata:
        labels:
          resource-type: cost-management
      spec:
        forProvider:
          amount: 1000
          timeGrain: Monthly
          timePeriod:
          - startDate: "2024-01-01T00:00:00Z"
          notification:
          - enabled: true
            threshold: 80
            operator: GreaterThan
            contactEmails: []
          subscriptionIdSelector:
            matchControllerRef: true
        providerConfigRef:
          name: azure-provider-prod
    
    patches:
    # Budget amount and currency
    - type: FromCompositeFieldPath
      fromFieldPath: spec.parameters.budget.amount
      toFieldPath: spec.forProvider.amount
    
    - type: FromCompositeFieldPath
      fromFieldPath: spec.parameters.budget.currency
      toFieldPath: spec.forProvider.currency
    
    # Dynamic notifications based on thresholds
    - type: FromCompositeFieldPath
      fromFieldPath: spec.parameters.budget.alertThresholds
      toFieldPath: spec.forProvider.notification
      transforms:
      - type: convert
        convert:
          toType: array
          format: json
    
    # Contact emails
    - type: FromCompositeFieldPath
      fromFieldPath: spec.parameters.securityContact.email
      toFieldPath: spec.forProvider.notification[0].contactEmails[0]
    
    # Conditional budget creation
    - type: FromCompositeFieldPath
      fromFieldPath: spec.parameters.budget.enabled
      toFieldPath: spec.forProvider.enabled
    
    - type: ToCompositeFieldPath
      fromFieldPath: status.atProvider.currentSpend
      toFieldPath: status.budgetStatus.currentSpend
    
    - type: ToCompositeFieldPath
      fromFieldPath: status.atProvider.forecastSpend
      toFieldPath: status.budgetStatus.forecastSpend
  
  # Role Assignment - Owners
  - name: rbac-owners
    base:
      apiVersion: authorization.azure.upbound.io/v1beta1
      kind: RoleAssignment
      metadata:
        labels:
          resource-type: rbac
          role: owner
      spec:
        forProvider:
          roleDefinitionId: "/providers/Microsoft.Authorization/roleDefinitions/8e3af657-a8ff-443c-a75c-2fe8c4bcb635"
          principalId: placeholder
          scope: placeholder
        providerConfigRef:
          name: azure-provider-prod
    
    patches:
    # Scope to subscription
    - type: CombineFromComposite
      combine:
        variables:
        - fromFieldPath: status.subscriptionId
        strategy: string
        string:
          fmt: "/subscriptions/%s"
      toFieldPath: spec.forProvider.scope
    
    # Principal IDs from owners list
    - type: FromCompositeFieldPath
      fromFieldPath: spec.parameters.rbac.owners
      toFieldPath: spec.forProvider.principalIds
      transforms:
      - type: convert
        convert:
          toType: array
    
    - type: FromCompositeFieldPath
      fromFieldPath: spec.resourceConfig.providerConfigName
      toFieldPath: spec.providerConfigRef.name
  
  # Role Assignment - Contributors
  - name: rbac-contributors
    base:
      apiVersion: authorization.azure.upbound.io/v1beta1
      kind: RoleAssignment
      metadata:
        labels:
          resource-type: rbac
          role: contributor
      spec:
        forProvider:
          roleDefinitionId: "/providers/Microsoft.Authorization/roleDefinitions/b24988ac-6180-42a0-ab88-20f7382dd24c"
          principalId: placeholder
          scope: placeholder
        providerConfigRef:
          name: azure-provider-prod
    
    patches:
    - type: CombineFromComposite
      combine:
        variables:
        - fromFieldPath: status.subscriptionId
        strategy: string
        string:
          fmt: "/subscriptions/%s"
      toFieldPath: spec.forProvider.scope
    
    - type: FromCompositeFieldPath
      fromFieldPath: spec.parameters.rbac.contributors
      toFieldPath: spec.forProvider.principalIds
      transforms:
      - type: convert
        convert:
          toType: array
  
  # Role Assignment - Readers
  - name: rbac-readers
    base:
      apiVersion: authorization.azure.upbound.io/v1beta1
      kind: RoleAssignment
      metadata:
        labels:
          resource-type: rbac
          role: reader
      spec:
        forProvider:
          roleDefinitionId: "/providers/Microsoft.Authorization/roleDefinitions/acdd72a7-3385-48ef-bd42-f606fba81ae7"
          principalId: placeholder
          scope: placeholder
        providerConfigRef:
          name: azure-provider-prod
    
    patches:
    - type: CombineFromComposite
      combine:
        variables:
        - fromFieldPath: status.subscriptionId
        strategy: string
        string:
          fmt: "/subscriptions/%s"
      toFieldPath: spec.forProvider.scope
    
    - type: FromCompositeFieldPath
      fromFieldPath: spec.parameters.rbac.readers
      toFieldPath: spec.forProvider.principalIds
      transforms:
      - type: convert
        convert:
          toType: array
  
  # Azure Policy Assignment - Required Tags
  - name: policy-required-tags
    base:
      apiVersion: authorization.azure.upbound.io/v1beta1
      kind: SubscriptionPolicyAssignment
      metadata:
        labels:
          resource-type: policy
          policy-type: governance
      spec:
        forProvider:
          policyDefinitionId: "/providers/Microsoft.Authorization/policyDefinitions/96670d01-0a4d-4649-9c89-2d3abc0a5025"
          displayName: "Require specific tags"
          description: "Enforces required tags on resources"
          parameters: |
            {
              "tagNames": {
                "value": ["environment", "costCenter", "owner"]
              }
            }
          subscriptionIdSelector:
            matchControllerRef: true
        providerConfigRef:
          name: azure-provider-prod
    
    patches:
    - type: FromCompositeFieldPath
      fromFieldPath: spec.parameters.policies.enabled
      toFieldPath: spec.forProvider.enforcementMode
      transforms:
      - type: map
        map:
          true: Default
          false: DoNotEnforce
    
    - type: FromCompositeFieldPath
      fromFieldPath: spec.resourceConfig.providerConfigName
      toFieldPath: spec.providerConfigRef.name
  
  # Azure Policy Assignment - Allowed Locations
  - name: policy-allowed-locations
    base:
      apiVersion: authorization.azure.upbound.io/v1beta1
      kind: SubscriptionPolicyAssignment
      metadata:
        labels:
          resource-type: policy
          policy-type: compliance
      spec:
        forProvider:
          policyDefinitionId: "/providers/Microsoft.Authorization/policyDefinitions/e56962a6-4747-49cd-b67b-bf8b01975c4c"
          displayName: "Allowed Azure regions"
          description: "Restricts resources to approved regions"
          subscriptionIdSelector:
            matchControllerRef: true
        providerConfigRef:
          name: azure-provider-prod
    
    patches:
    - type: FromCompositeFieldPath
      fromFieldPath: spec.parameters.networking.allowedRegions
      toFieldPath: spec.forProvider.parameters
      transforms:
      - type: string
        string:
          fmt: '{"listOfAllowedLocations": {"value": %s}}'
          type: Format
    
    - type: FromCompositeFieldPath
      fromFieldPath: spec.parameters.policies.enabled
      toFieldPath: spec.forProvider.enforcementMode
      transforms:
      - type: map
        map:
          true: Default
          false: DoNotEnforce
  
  # Security Center Contact
  - name: security-contact
    base:
      apiVersion: security.azure.upbound.io/v1beta1
      kind: SecurityCenterContact
      metadata:
        labels:
          resource-type: security
      spec:
        forProvider:
          email: placeholder
          phone: placeholder
          alertNotifications: true
          alertsToAdmins: true
          subscriptionIdSelector:
            matchControllerRef: true
        providerConfigRef:
          name: azure-provider-prod
    
    patches:
    - type: FromCompositeFieldPath
      fromFieldPath: spec.parameters.securityContact.email
      toFieldPath: spec.forProvider.email
    
    - type: FromCompositeFieldPath
      fromFieldPath: spec.parameters.securityContact.phone
      toFieldPath: spec.forProvider.phone
    
    - type: FromCompositeFieldPath
      fromFieldPath: spec.resourceConfig.providerConfigName
      toFieldPath: spec.providerConfigRef.name
  
  # Log Analytics Workspace (for monitoring)
  - name: log-analytics
    base:
      apiVersion: operationalinsights.azure.upbound.io/v1beta1
      kind: Workspace
      metadata:
        labels:
          resource-type: monitoring
      spec:
        forProvider:
          location: westeurope
          sku: PerGB2018
          retentionInDays: 90
          resourceGroupName: platform-monitoring
        providerConfigRef:
          name: azure-provider-prod
    
    patches:
    - type: FromCompositeFieldPath
      fromFieldPath: spec.parameters.networking.allowedRegions[0]
      toFieldPath: spec.forProvider.location
    
    - type: FromCompositeFieldPath
      fromFieldPath: spec.parameters.monitoring.retentionDays
      toFieldPath: spec.forProvider.retentionInDays
    
    - type: CombineFromComposite
      combine:
        variables:
        - fromFieldPath: spec.parameters.subscriptionName
        strategy: string
        string:
          fmt: "log-%s"
      toFieldPath: metadata.name
    
    - type: ToCompositeFieldPath
      fromFieldPath: status.atProvider.workspaceId
      toFieldPath: status.logAnalyticsWorkspaceId
  
  # Diagnostic Settings for Subscription
  - name: diagnostic-settings
    base:
      apiVersion: insights.azure.upbound.io/v1beta1
      kind: MonitorDiagnosticSetting
      metadata:
        labels:
          resource-type: monitoring
      spec:
        forProvider:
          logAnalyticsWorkspaceIdSelector:
            matchControllerRef: true
            matchLabels:
              resource-type: monitoring
          log:
          - category: Administrative
            enabled: true
          - category: Security
            enabled: true
          - category: ServiceHealth
            enabled: true
          - category: Alert
            enabled: true
          - category: Recommendation
            enabled: true
          - category: Policy
            enabled: true
          - category: Autoscale
            enabled: true
          - category: ResourceHealth
            enabled: true
          targetResourceIdSelector:
            matchControllerRef: true
        providerConfigRef:
          name: azure-provider-prod
    
    patches:
    - type: CombineFromComposite
      combine:
        variables:
        - fromFieldPath: status.subscriptionId
        strategy: string
        string:
          fmt: "/subscriptions/%s"
      toFieldPath: spec.forProvider.targetResourceId
```

## Using yq to Manipulate the Composition

### Update Provider Config for All Resources

```bash
yq -i '
  .spec.resources[].base.spec.providerConfigRef.name = "azure-provider-new"
' azure-subscription-composition.yaml
```

### Add New Resource

```bash
# Add Defender for Cloud configuration
yq -i '
  .spec.resources += [{
    "name": "defender-for-cloud",
    "base": {
      "apiVersion": "security.azure.upbound.io/v1beta1",
      "kind": "SecurityCenterSubscriptionPricing",
      "metadata": {
        "labels": {
          "resource-type": "security"
        }
      },
      "spec": {
        "forProvider": {
          "tier": "Standard",
          "resourceType": "VirtualMachines"
        },
        "providerConfigRef": {
          "name": "azure-provider-prod"
        }
      }
    },
    "patches": [
      {
        "type": "FromCompositeFieldPath",
        "fromFieldPath": "spec.parameters.defenderForCloud.tier",
        "toFieldPath": "spec.forProvider.tier"
      }
    ]
  }]
' azure-subscription-composition.yaml
```

### Update Budget Thresholds

```bash
# Change default budget notification threshold
yq -i '
  (.spec.resources[] | 
   select(.name == "budget").base.spec.forProvider.notification[0].threshold) = 75
' azure-subscription-composition.yaml
```

### Add Patch to Existing Resource

```bash
# Add location patch to log analytics
yq -i '
  (.spec.resources[] | 
   select(.name == "log-analytics").patches) += [{
    "type": "FromCompositeFieldPath",
    "fromFieldPath": "spec.parameters.tags",
    "toFieldPath": "spec.forProvider.tags"
  }]
' azure-subscription-composition.yaml
```

### Update Role Definition IDs

```bash
# Update contributor role definition
yq -i '
  (.spec.resources[] | 
   select(.name == "rbac-contributors").base.spec.forProvider.roleDefinitionId) = 
  "/providers/Microsoft.Authorization/roleDefinitions/b24988ac-6180-42a0-ab88-20f7382dd24c"
' azure-subscription-composition.yaml
```

### Add Connection Details

```bash
# Add client credentials to connection details
yq -i '
  (.spec.resources[] | 
   select(.name == "subscription").connectionDetails) += [
    {
      "name": "clientId",
      "fromFieldPath": "status.atProvider.clientId"
    },
    {
      "name": "clientSecret",
      "fromConnectionSecretKey": "clientSecret"
    }
  ]
' azure-subscription-composition.yaml
```

### Modify Policy Parameters

```bash
# Update allowed locations policy parameters
yq -i '
  (.spec.resources[] | 
   select(.name == "policy-allowed-locations").base.spec.forProvider.parameters) = 
  "{\"listOfAllowedLocations\": {\"value\": [\"westeurope\", \"northeurope\", \"uksouth\"]}}"
' azure-subscription-composition.yaml
```

### Add Readiness Check

```bash
# Add readiness check to budget resource
yq -i '
  (.spec.resources[] | 
   select(.name == "budget").readinessChecks) = [{
    "type": "MatchString",
    "fieldPath": "status.atProvider.state",
    "matchString": "Active"
  }]
' azure-subscription-composition.yaml
```

### Update Labels

```bash
# Add compliance label to all resources
yq -i '
  .spec.resources[].base.metadata.labels.compliance = "iso27001"
' azure-subscription-composition.yaml
```

### Extract Resource Configuration

```bash
# List all resource names and kinds
yq '.spec.resources[] | {name: .name, kind: .base.kind}' azure-subscription-composition.yaml

# Get all patch configurations
yq '.spec.resources[] | select(.patches != null) | {name: .name, patches: .patches | length}' azure-subscription-composition.yaml

# List all provider config references
yq '.spec.resources[].base.spec.providerConfigRef.name' azure-subscription-composition.yaml
```

## Environment-Specific Variants

### Create Development Variant

```bash
yq '
  .metadata.name = "azure-subscription-dev" |
  .metadata.labels.tier = "basic" |
  .metadata.labels.environment = "development" |
  
  # Reduce budget
  (.spec.resources[] | 
   select(.name == "budget").base.spec.forProvider.amount) = 1000 |
  
  # Disable policy enforcement
  (.spec.resources[] | 
   select(.metadata.labels."resource-type" == "policy").base.spec.forProvider.enforcementMode) = "DoNotEnforce" |
  
  # Use basic monitoring
  (.spec.resources[] | 
   select(.name == "log-analytics").base.spec.forProvider.sku) = "Free" |
  (.spec.resources[] | 
   select(.name == "log-analytics").base.spec.forProvider.retentionInDays) = 30
' azure-subscription-composition.yaml > azure-subscription-composition-dev.yaml
```

### Create Production Variant

```bash
yq '
  .metadata.name = "azure-subscription-production" |
  .metadata.labels.tier = "premium" |
  .metadata.labels.environment = "production" |
  
  # Increase budget
  (.spec.resources[] | 
   select(.name == "budget").base.spec.forProvider.amount) = 50000 |
  
  # Enforce all policies
  (.spec.resources[] | 
   select(.metadata.labels."resource-type" == "policy").base.spec.forProvider.enforcementMode) = "Default" |
  
  # Extended retention
  (.spec.resources[] | 
   select(.name == "log-analytics").base.spec.forProvider.retentionInDays) = 365
' azure-subscription-composition.yaml > azure-subscription-composition-prod.yaml
```

## Automation Scripts

### Bulk Update Composition

```bash
#!/bin/bash
# update-subscription-compositions.sh

COMPOSITIONS_DIR="./compositions/subscriptions"
NEW_PROVIDER="azure-provider-v2"
NEW_LOG_RETENTION=180

for comp in "$COMPOSITIONS_DIR"/*.yaml; do
  echo "Updating $comp..."
  
  # Update provider config
  yq -i "
    .spec.resources[].base.spec.providerConfigRef.name = \"$NEW_PROVIDER\"
  " "$comp"
  
  # Update log retention
  yq -i "
    (.spec.resources[] | 
     select(.name == \"log-analytics\").base.spec.forProvider.retentionInDays) = $NEW_LOG_RETENTION
  " "$comp"
  
  # Validate
  if ! yq '.' "$comp" > /dev/null 2>&1; then
    echo "ERROR: Invalid YAML in $comp"
    exit 1
  fi
  
  echo "✓ Updated $comp"
done
```

### Validate Composition

```bash
#!/bin/bash
# validate-composition.sh

COMP_FILE="$1"

echo "Validating $COMP_FILE..."

# Check basic structure
if [ "$(yq '.kind' "$COMP_FILE")" != "Composition" ]; then
  echo "ERROR: Not a Composition"
  exit 1
fi

# Check all resources have names
unnamed=$(yq '.spec.resources[] | select(.name == null)' "$COMP_FILE")
if [ -n "$unnamed" ]; then
  echo "ERROR: Found resources without names"
  exit 1
fi

# Check all resources have provider configs
missing_provider=$(yq '.spec.resources[] | 
  select(.base.spec.providerConfigRef == null) | 
  .name' "$COMP_FILE")

if [ -n "$missing_provider" ]; then
  echo "ERROR: Resources missing provider config: $missing_provider"
  exit 1
fi

# Check compositeTypeRef matches XRD
composite_ref=$(yq '.spec.compositeTypeRef.kind' "$COMP_FILE")
if [ "$composite_ref" != "XAzureSubscription" ]; then
  echo "WARNING: compositeTypeRef kind is $composite_ref, expected XAzureSubscription"
fi

echo "✓ Composition is valid"
```

### Generate Documentation

```bash
#!/bin/bash
# generate-composition-docs.sh

COMP_FILE="azure-subscription-composition.yaml"
OUTPUT="COMPOSITION.md"

cat > "$OUTPUT" << EOF
# Azure Subscription Composition

## Resources

$(yq '.spec.resources[] | "### " + .name + "\n\n" +
  "**Kind**: " + .base.kind + "\n" +
  "**API Version**: " + .base.apiVersion + "\n" +
  "**Labels**: " + (.base.metadata.labels | to_entries | map(.key + "=" + .value) | join(", ")) + "\n\n" +
  "**Patches**: " + (if .patches then (.patches | length | tostring) else "0" end) + "\n\n"' "$COMP_FILE")

## Patch Summary

$(yq '.spec.resources[] | 
  select(.patches != null) | 
  "### " + .name + "\n\n" +
  (.patches[] | "- **" + .type + "**: " + .fromFieldPath + " → " + .toFieldPath + "\n") + "\n"' "$COMP_FILE")
EOF

echo "Generated: $OUTPUT"
```

## Testing

### Test Composition with Sample Claim

```bash
# Create test claim
cat > test-claim.yaml << EOF
apiVersion: platform.example.com/v1alpha1
kind: AzureSubscription
metadata:
  name: test-subscription
spec:
  parameters:
    subscriptionName: test-sub
    environment: dev
    costCenter: CC-9999
    billingAccount: "/test/billing"
    securityContact:
      email: test@example.com
    budget:
      amount: 1000
      currency: EUR
    rbac:
      owners: ["test@example.com"]
EOF

# Validate it would work with composition
kubectl crossplane beta validate test-claim.yaml \
  --composition azure-subscription-composition.yaml \
  --xrd azure-subscription-xrd.yaml
```

## Best Practices

1. **Resource Naming**: Use descriptive names for resources (`rbac-owners`, not `resource1`)
1. **Labels**: Add consistent labels for filtering and organization
1. **Patches**: Use specific patch types and include transforms where needed
1. **Connection Details**: Expose necessary credentials securely
1. **Readiness Checks**: Validate resource readiness before considering composition ready
1. **Provider Configs**: Reference provider configs consistently
1. **Status Patches**: Surface important status information to composite resource
1. **Documentation**: Keep annotations up to date

## Next Steps

- [XRD Definition](./azure-subscription-xrd.md)
- [Provider Configuration](../05-crossplane/03-provider-configs.md)
- [Patching Strategies](../05-crossplane/04-patches.md)

This Composition provides enterprise-grade Azure subscription provisioning with governance, security, and cost management through Crossplane.
