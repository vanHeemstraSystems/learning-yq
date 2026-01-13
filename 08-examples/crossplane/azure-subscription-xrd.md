# Example: Azure Subscription XRD (Composite Resource Definition)

This example demonstrates a Crossplane Composite Resource Definition (XRD) for Azure Subscriptions, which defines the API schema for requesting Azure subscriptions in a platform engineering context.

## XRD File

**File: azure-subscription-xrd.yaml**

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xazuresubscriptions.platform.example.com
  annotations:
    crossplane.io/description: "Azure Subscription with governance and policies"
    crossplane.io/version: "1.0.0"
spec:
  group: platform.example.com
  
  names:
    kind: XAzureSubscription
    plural: xazuresubscriptions
    singular: xazuresubscription
    shortNames:
    - xazuresub
    - xsub
  
  claimNames:
    kind: AzureSubscription
    plural: azuresubscriptions
    singular: azuresubscription
  
  connectionSecretKeys:
  - subscriptionId
  - tenantId
  - clientId
  - clientSecret
  
  versions:
  - name: v1alpha1
    served: true
    referenceable: true
    
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            description: "Specification for Azure Subscription"
            
            properties:
              # Basic subscription parameters
              parameters:
                type: object
                description: "Subscription configuration parameters"
                
                properties:
                  # Subscription details
                  subscriptionName:
                    type: string
                    description: "Name of the Azure subscription"
                    pattern: "^[a-zA-Z0-9-_]{3,64}$"
                  
                  displayName:
                    type: string
                    description: "Display name for the subscription"
                  
                  # Billing and management
                  billingAccount:
                    type: string
                    description: "Billing account ID"
                  
                  managementGroup:
                    type: string
                    description: "Management group to assign subscription to"
                    default: "platform-subscriptions"
                  
                  # Environment classification
                  environment:
                    type: string
                    description: "Environment type"
                    enum:
                    - dev
                    - test
                    - staging
                    - production
                    default: dev
                  
                  # Cost management
                  costCenter:
                    type: string
                    description: "Cost center for billing"
                  
                  budget:
                    type: object
                    description: "Budget configuration"
                    properties:
                      enabled:
                        type: boolean
                        default: true
                      amount:
                        type: integer
                        description: "Budget amount in currency"
                        minimum: 100
                      currency:
                        type: string
                        default: "EUR"
                        enum:
                        - EUR
                        - USD
                        - GBP
                      alertThresholds:
                        type: array
                        description: "Alert at these percentage thresholds"
                        default: [50, 80, 90, 100]
                        items:
                          type: integer
                          minimum: 1
                          maximum: 100
                    required:
                    - amount
                  
                  # Governance
                  tags:
                    type: object
                    description: "Resource tags"
                    additionalProperties:
                      type: string
                    default:
                      managed-by: crossplane
                  
                  # Security and compliance
                  securityContact:
                    type: object
                    description: "Security contact information"
                    properties:
                      email:
                        type: string
                        format: email
                      phone:
                        type: string
                    required:
                    - email
                  
                  # Azure Policy assignments
                  policies:
                    type: object
                    description: "Azure Policy configuration"
                    properties:
                      enabled:
                        type: boolean
                        default: true
                      assignments:
                        type: array
                        description: "Policy assignments to apply"
                        items:
                          type: object
                          properties:
                            name:
                              type: string
                            policyDefinitionId:
                              type: string
                            parameters:
                              type: object
                              x-kubernetes-preserve-unknown-fields: true
                  
                  # Networking defaults
                  networking:
                    type: object
                    description: "Default networking configuration"
                    properties:
                      allowedRegions:
                        type: array
                        description: "Allowed Azure regions"
                        default:
                        - westeurope
                        - northeurope
                        items:
                          type: string
                      
                      hubVnetId:
                        type: string
                        description: "Hub VNet ID for connectivity"
                  
                  # Role-based access control
                  rbac:
                    type: object
                    description: "RBAC configuration"
                    properties:
                      owners:
                        type: array
                        description: "Subscription owners"
                        items:
                          type: string
                      contributors:
                        type: array
                        description: "Subscription contributors"
                        items:
                          type: string
                      readers:
                        type: array
                        description: "Subscription readers"
                        items:
                          type: string
                
                required:
                - subscriptionName
                - environment
                - costCenter
                - securityContact
              
              # Composition selector
              compositionSelector:
                type: object
                properties:
                  matchLabels:
                    type: object
                    additionalProperties:
                      type: string
              
              compositionRef:
                type: object
                properties:
                  name:
                    type: string
              
              # Resource configuration
              resourceConfig:
                type: object
                description: "Resource configuration options"
                properties:
                  providerConfigName:
                    type: string
                    description: "Crossplane provider config to use"
                    default: "azure-provider-prod"
                  
                  deletionPolicy:
                    type: string
                    description: "Resource deletion policy"
                    enum:
                    - Delete
                    - Orphan
                    default: Delete
            
            required:
            - parameters
          
          # Status fields
          status:
            type: object
            properties:
              subscriptionId:
                type: string
                description: "Azure subscription ID"
              
              tenantId:
                type: string
                description: "Azure tenant ID"
              
              state:
                type: string
                description: "Subscription provisioning state"
              
              billingScope:
                type: string
                description: "Billing scope ID"
              
              managementGroupId:
                type: string
                description: "Management group assignment"
              
              budgetStatus:
                type: object
                description: "Budget status"
                properties:
                  currentSpend:
                    type: number
                  percentageUsed:
                    type: number
                  lastUpdated:
                    type: string
                    format: date-time
              
              policyCompliance:
                type: object
                description: "Policy compliance status"
                properties:
                  compliant:
                    type: boolean
                  nonCompliantPolicies:
                    type: array
                    items:
                      type: string
              
              conditions:
                type: array
                description: "Resource conditions"
                items:
                  type: object
                  properties:
                    type:
                      type: string
                    status:
                      type: string
                    lastTransitionTime:
                      type: string
                      format: date-time
                    reason:
                      type: string
                    message:
                      type: string
    
    additionalPrinterColumns:
    - name: Environment
      type: string
      jsonPath: .spec.parameters.environment
    - name: Subscription ID
      type: string
      jsonPath: .status.subscriptionId
    - name: State
      type: string
      jsonPath: .status.state
    - name: Budget
      type: integer
      jsonPath: .spec.parameters.budget.amount
    - name: Compliant
      type: boolean
      jsonPath: .status.policyCompliance.compliant
    - name: Age
      type: date
      jsonPath: .metadata.creationTimestamp
```

## Using yq to Manipulate the XRD

### Add New Parameter Field

```bash
# Add a new parameter for Defender for Cloud
yq -i '
  .spec.versions[0].schema.openAPIV3Schema.properties.spec.properties.parameters.properties.defenderForCloud = {
    "type": "object",
    "description": "Microsoft Defender for Cloud configuration",
    "properties": {
      "enabled": {
        "type": "boolean",
        "default": true
      },
      "tier": {
        "type": "string",
        "enum": ["Free", "Standard"],
        "default": "Standard"
      }
    }
  }
' azure-subscription-xrd.yaml
```

### Update Version

```bash
# Change version from v1alpha1 to v1beta1
yq -i '
  .spec.versions[0].name = "v1beta1" |
  .metadata.annotations."crossplane.io/version" = "1.1.0"
' azure-subscription-xrd.yaml
```

### Add New Enum Value

```bash
# Add 'pre-production' to environment enum
yq -i '
  .spec.versions[0].schema.openAPIV3Schema.properties.spec.properties.parameters.properties.environment.enum += ["pre-production"]
' azure-subscription-xrd.yaml
```

### Add Connection Secret Key

```bash
# Add new connection secret key
yq -i '
  .spec.connectionSecretKeys += ["managementEndpoint"]
' azure-subscription-xrd.yaml
```

### Add Printer Column

```bash
# Add cost center as a printer column
yq -i '
  .spec.versions[0].additionalPrinterColumns += [{
    "name": "Cost Center",
    "type": "string",
    "jsonPath": ".spec.parameters.costCenter"
  }]
' azure-subscription-xrd.yaml
```

### Update Default Values

```bash
# Change default budget alert thresholds
yq -i '
  .spec.versions[0].schema.openAPIV3Schema.properties.spec.properties.parameters.properties.budget.properties.alertThresholds.default = [60, 85, 95, 100]
' azure-subscription-xrd.yaml
```

### Add Validation Pattern

```bash
# Add pattern validation for cost center
yq -i '
  .spec.versions[0].schema.openAPIV3Schema.properties.spec.properties.parameters.properties.costCenter.pattern = "^CC-[0-9]{4}$"
' azure-subscription-xrd.yaml
```

### Add New Security Parameter

```bash
# Add security monitoring configuration
yq -i '
  .spec.versions[0].schema.openAPIV3Schema.properties.spec.properties.parameters.properties.monitoring = {
    "type": "object",
    "description": "Monitoring and alerting configuration",
    "properties": {
      "logAnalyticsWorkspaceId": {
        "type": "string",
        "description": "Log Analytics workspace ID"
      },
      "retentionDays": {
        "type": "integer",
        "default": 90,
        "minimum": 30,
        "maximum": 730
      },
      "enableSentinel": {
        "type": "boolean",
        "default": false
      }
    }
  }
' azure-subscription-xrd.yaml
```

### Extract Schema for Documentation

```bash
# Extract all parameter names and descriptions
yq '
  .spec.versions[0].schema.openAPIV3Schema.properties.spec.properties.parameters.properties |
  to_entries |
  map({
    parameter: .key,
    description: .value.description,
    type: .value.type,
    required: (.value.required // false)
  })
' azure-subscription-xrd.yaml
```

### Validate Required Fields

```bash
# Check which parameters are required
yq '
  .spec.versions[0].schema.openAPIV3Schema.properties.spec.properties.parameters.required[]
' azure-subscription-xrd.yaml
```

### Add Status Field

```bash
# Add new status field for compliance score
yq -i '
  .spec.versions[0].schema.openAPIV3Schema.properties.status.properties.complianceScore = {
    "type": "integer",
    "description": "Overall compliance score (0-100)",
    "minimum": 0,
    "maximum": 100
  }
' azure-subscription-xrd.yaml
```

### Environment-Specific Customization

```bash
# Create production-focused variant
yq '
  .metadata.name = "xazuresubscriptions-prod.platform.example.com" |
  .spec.versions[0].schema.openAPIV3Schema.properties.spec.properties.parameters.properties.environment.default = "production" |
  .spec.versions[0].schema.openAPIV3Schema.properties.spec.properties.budget.properties.amount.minimum = 1000 |
  .spec.versions[0].schema.openAPIV3Schema.properties.spec.properties.policies.properties.enabled.default = true
' azure-subscription-xrd.yaml > azure-subscription-xrd-prod.yaml
```

### Audit XRD Configuration

```bash
# List all enum fields
yq '
  .spec.versions[0].schema.openAPIV3Schema.properties.spec.properties.parameters.properties |
  to_entries |
  map(select(.value.enum != null) | {
    field: .key,
    values: .value.enum
  })
' azure-subscription-xrd.yaml

# List all default values
yq '
  .spec.versions[0].schema.openAPIV3Schema.. | 
  select(has("default")) |
  {path: path | join("."), default: .default}
' azure-subscription-xrd.yaml
```

### Generate TypeScript Interface

```bash
# Extract schema to help generate TypeScript types
yq -o json '
  .spec.versions[0].schema.openAPIV3Schema.properties.spec.properties.parameters.properties |
  to_entries |
  map({
    name: .key,
    type: .value.type,
    description: .value.description
  })
' azure-subscription-xrd.yaml
```

## Example Claim Using This XRD

**File: my-subscription-claim.yaml**

```yaml
apiVersion: platform.example.com/v1alpha1
kind: AzureSubscription
metadata:
  name: project-phoenix-prod
  namespace: platform-team
spec:
  parameters:
    subscriptionName: project-phoenix-production
    displayName: "Project Phoenix - Production"
    environment: production
    costCenter: CC-1234
    
    billingAccount: "/providers/Microsoft.Billing/billingAccounts/12345678"
    managementGroup: "production-workloads"
    
    budget:
      enabled: true
      amount: 5000
      currency: EUR
      alertThresholds: [50, 75, 90, 100]
    
    tags:
      project: phoenix
      owner: platform-team
      environment: production
      compliance: iso27001
    
    securityContact:
      email: security@example.com
      phone: "+31-20-1234567"
    
    policies:
      enabled: true
      assignments:
      - name: require-tags
        policyDefinitionId: "/providers/Microsoft.Authorization/policyDefinitions/require-tags"
        parameters:
          tagNames:
          - environment
          - costCenter
      
      - name: allowed-locations
        policyDefinitionId: "/providers/Microsoft.Authorization/policyDefinitions/allowed-locations"
        parameters:
          listOfAllowedLocations:
          - westeurope
          - northeurope
    
    networking:
      allowedRegions:
      - westeurope
      - northeurope
      hubVnetId: "/subscriptions/hub-sub-id/resourceGroups/networking/providers/Microsoft.Network/virtualNetworks/hub-vnet"
    
    rbac:
      owners:
      - "platform-team@example.com"
      contributors:
      - "project-phoenix-team@example.com"
      readers:
      - "finance-team@example.com"
  
  compositionSelector:
    matchLabels:
      provider: azure
      tier: enterprise
  
  resourceConfig:
    providerConfigName: azure-provider-prod
    deletionPolicy: Delete

  writeConnectionSecretToRef:
    name: project-phoenix-prod-credentials
    namespace: platform-team
```

### Manipulating the Claim with yq

```bash
# Update budget amount
yq -i '.spec.parameters.budget.amount = 10000' my-subscription-claim.yaml

# Add new tag
yq -i '.spec.parameters.tags.dataClassification = "confidential"' my-subscription-claim.yaml

# Add RBAC assignment
yq -i '.spec.parameters.rbac.readers += ["audit-team@example.com"]' my-subscription-claim.yaml

# Change environment
yq -i '.spec.parameters.environment = "staging"' my-subscription-claim.yaml

# Add policy assignment
yq -i '
  .spec.parameters.policies.assignments += [{
    "name": "require-encryption",
    "policyDefinitionId": "/providers/Microsoft.Authorization/policyDefinitions/require-storage-encryption"
  }]
' my-subscription-claim.yaml
```

## Validation Scripts

### Validate XRD Structure

```bash
#!/bin/bash
# validate-xrd.sh

XRD_FILE="azure-subscription-xrd.yaml"

echo "Validating XRD structure..."

# Check API version
if ! yq '.apiVersion' "$XRD_FILE" | grep -q "apiextensions.crossplane.io"; then
  echo "ERROR: Invalid apiVersion"
  exit 1
fi

# Check kind
if [ "$(yq '.kind' "$XRD_FILE")" != "CompositeResourceDefinition" ]; then
  echo "ERROR: Kind must be CompositeResourceDefinition"
  exit 1
fi

# Check required spec fields
required_fields=("group" "names" "versions")
for field in "${required_fields[@]}"; do
  if ! yq -e ".spec.$field" "$XRD_FILE" > /dev/null 2>&1; then
    echo "ERROR: Missing required field: spec.$field"
    exit 1
  fi
done

# Validate versions have schema
if ! yq -e '.spec.versions[0].schema.openAPIV3Schema' "$XRD_FILE" > /dev/null 2>&1; then
  echo "ERROR: Version must have schema"
  exit 1
fi

echo "✓ XRD structure is valid"
```

### Extract Documentation

```bash
#!/bin/bash
# generate-xrd-docs.sh

XRD_FILE="azure-subscription-xrd.yaml"
OUTPUT="PARAMETERS.md"

cat > "$OUTPUT" << 'EOF'
# Azure Subscription Parameters

## Available Parameters

EOF

yq '
  .spec.versions[0].schema.openAPIV3Schema.properties.spec.properties.parameters.properties |
  to_entries |
  map("### " + .key + "\n\n" +
      (.value.description // "No description") + "\n\n" +
      "- **Type**: " + .value.type + "\n" +
      (if .value.default then "- **Default**: `" + (.value.default | tostring) + "`\n" else "" end) +
      (if .value.enum then "- **Allowed values**: " + (.value.enum | join(", ")) + "\n" else "" end) +
      "\n")
' "$XRD_FILE" >> "$OUTPUT"

echo "Generated documentation: $OUTPUT"
```

## Best Practices

1. **Version Management**: Always update the version annotation when modifying the schema
1. **Backwards Compatibility**: Add new fields as optional to maintain compatibility
1. **Validation**: Use patterns and enums to validate input
1. **Documentation**: Keep descriptions clear and comprehensive
1. **Defaults**: Provide sensible defaults for optional fields
1. **Required Fields**: Minimize required fields to improve user experience
1. **Status Fields**: Include useful status information for operators

## Related Files

- **Composition**: `azure-subscription-composition.yaml` (implements this XRD)
- **Provider Config**: Azure provider configuration for authentication
- **Claims**: User-facing subscription requests

This XRD provides a comprehensive API for Azure subscription management through Crossplane, suitable for enterprise platform engineering.
