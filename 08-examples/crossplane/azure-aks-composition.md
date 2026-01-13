# Example: Azure AKS Cluster Composition

This is a practical example of a Crossplane Composition for deploying an Azure Kubernetes Service (AKS) cluster with supporting resources.

## Composition File

**File: azure-aks-composition.yaml**

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: azure-aks-cluster
  labels:
    provider: azure
    service: kubernetes
    environment: production
  annotations:
    crossplane.io/description: "Azure AKS cluster with managed identity and networking"
    crossplane.io/version: "1.0.0"
spec:
  writeConnectionSecretsToNamespace: crossplane-system
  
  compositeTypeRef:
    apiVersion: platform.example.com/v1alpha1
    kind: XKubernetesCluster
  
  resources:
  # Resource Group
  - name: resourcegroup
    base:
      apiVersion: azure.upbound.io/v1beta1
      kind: ResourceGroup
      metadata:
        labels:
          resource-type: infrastructure
      spec:
        forProvider:
          location: westeurope
        providerConfigRef:
          name: azure-provider-prod
    patches:
    - type: FromCompositeFieldPath
      fromFieldPath: spec.parameters.location
      toFieldPath: spec.forProvider.location
    - type: FromCompositeFieldPath
      fromFieldPath: metadata.name
      toFieldPath: spec.forProvider.name
      transforms:
      - type: string
        string:
          fmt: "rg-%s"
    - type: ToCompositeFieldPath
      fromFieldPath: status.atProvider.id
      toFieldPath: status.resourceGroupId
  
  # Virtual Network
  - name: virtualnetwork
    base:
      apiVersion: network.azure.upbound.io/v1beta1
      kind: VirtualNetwork
      metadata:
        labels:
          resource-type: networking
      spec:
        forProvider:
          location: westeurope
          addressSpace:
          - 10.0.0.0/8
          resourceGroupNameSelector:
            matchControllerRef: true
    patches:
    - type: FromCompositeFieldPath
      fromFieldPath: spec.parameters.location
      toFieldPath: spec.forProvider.location
    - type: FromCompositeFieldPath
      fromFieldPath: spec.parameters.vnetCidr
      toFieldPath: spec.forProvider.addressSpace[0]
    - type: FromCompositeFieldPath
      fromFieldPath: metadata.name
      toFieldPath: metadata.name
      transforms:
      - type: string
        string:
          fmt: "vnet-%s"
  
  # Subnet for AKS
  - name: subnet
    base:
      apiVersion: network.azure.upbound.io/v1beta1
      kind: Subnet
      metadata:
        labels:
          resource-type: networking
      spec:
        forProvider:
          addressPrefixes:
          - 10.240.0.0/16
          virtualNetworkNameSelector:
            matchControllerRef: true
          resourceGroupNameSelector:
            matchControllerRef: true
    patches:
    - type: FromCompositeFieldPath
      fromFieldPath: spec.parameters.subnetCidr
      toFieldPath: spec.forProvider.addressPrefixes[0]
    - type: FromCompositeFieldPath
      fromFieldPath: metadata.name
      toFieldPath: metadata.name
      transforms:
      - type: string
        string:
          fmt: "subnet-aks-%s"
    - type: ToCompositeFieldPath
      fromFieldPath: status.atProvider.id
      toFieldPath: status.subnetId
  
  # AKS Cluster
  - name: akscluster
    base:
      apiVersion: containerservice.azure.upbound.io/v1beta1
      kind: KubernetesCluster
      metadata:
        labels:
          resource-type: compute
      spec:
        forProvider:
          location: westeurope
          resourceGroupNameSelector:
            matchControllerRef: true
          dnsPrefix: aks
          kubernetesVersion: "1.28"
          
          defaultNodePool:
          - name: default
            nodeCount: 3
            vmSize: Standard_D2s_v3
            vnetSubnetIdSelector:
              matchControllerRef: true
            enableAutoScaling: true
            minCount: 1
            maxCount: 5
            osDiskSizeGb: 128
            type: VirtualMachineScaleSets
          
          identity:
          - type: SystemAssigned
          
          networkProfile:
          - networkPlugin: azure
            networkPolicy: azure
            serviceCidr: 10.0.0.0/16
            dnsServiceIp: 10.0.0.10
            dockerBridgeCidr: 172.17.0.1/16
            loadBalancerSku: standard
          
          addonProfile:
          - azurePolicy:
            - enabled: true
            omsAgent:
            - enabled: true
    
    patches:
    # Location
    - type: FromCompositeFieldPath
      fromFieldPath: spec.parameters.location
      toFieldPath: spec.forProvider.location
    
    # Cluster name
    - type: FromCompositeFieldPath
      fromFieldPath: metadata.name
      toFieldPath: metadata.name
      transforms:
      - type: string
        string:
          fmt: "aks-%s"
    
    # Kubernetes version
    - type: FromCompositeFieldPath
      fromFieldPath: spec.parameters.kubernetesVersion
      toFieldPath: spec.forProvider.kubernetesVersion
    
    # Node count
    - type: FromCompositeFieldPath
      fromFieldPath: spec.parameters.nodeCount
      toFieldPath: spec.forProvider.defaultNodePool[0].nodeCount
    
    # VM size
    - type: FromCompositeFieldPath
      fromFieldPath: spec.parameters.nodeSize
      toFieldPath: spec.forProvider.defaultNodePool[0].vmSize
      transforms:
      - type: map
        map:
          small: Standard_D2s_v3
          medium: Standard_D4s_v3
          large: Standard_D8s_v3
    
    # Auto-scaling
    - type: FromCompositeFieldPath
      fromFieldPath: spec.parameters.autoScaling.enabled
      toFieldPath: spec.forProvider.defaultNodePool[0].enableAutoScaling
    - type: FromCompositeFieldPath
      fromFieldPath: spec.parameters.autoScaling.minCount
      toFieldPath: spec.forProvider.defaultNodePool[0].minCount
    - type: FromCompositeFieldPath
      fromFieldPath: spec.parameters.autoScaling.maxCount
      toFieldPath: spec.forProvider.defaultNodePool[0].maxCount
    
    # Status updates
    - type: ToCompositeFieldPath
      fromFieldPath: status.atProvider.fqdn
      toFieldPath: status.clusterFqdn
    - type: ToCompositeFieldPath
      fromFieldPath: status.atProvider.kubernetesVersion
      toFieldPath: status.kubernetesVersion
    
    # Connection details
    connectionDetails:
    - name: kubeconfig
      fromConnectionSecretKey: kubeconfig
    
    # Readiness checks
    readinessChecks:
    - type: MatchString
      fieldPath: status.atProvider.provisioningState
      matchString: Succeeded
    - type: None
      fieldPath: status.conditions[?(@.type=='Ready')].status
      matchString: "False"
  
  # Log Analytics Workspace (for monitoring)
  - name: loganalytics
    base:
      apiVersion: operationalinsights.azure.upbound.io/v1beta1
      kind: Workspace
      metadata:
        labels:
          resource-type: monitoring
      spec:
        forProvider:
          location: westeurope
          resourceGroupNameSelector:
            matchControllerRef: true
          sku: PerGB2018
          retentionInDays: 30
    patches:
    - type: FromCompositeFieldPath
      fromFieldPath: spec.parameters.location
      toFieldPath: spec.forProvider.location
    - type: FromCompositeFieldPath
      fromFieldPath: metadata.name
      toFieldPath: metadata.name
      transforms:
      - type: string
        string:
          fmt: "log-%s"
    - type: FromCompositeFieldPath
      fromFieldPath: spec.parameters.logRetentionDays
      toFieldPath: spec.forProvider.retentionInDays
    - type: ToCompositeFieldPath
      fromFieldPath: status.atProvider.workspaceId
      toFieldPath: status.logAnalyticsWorkspaceId
```

## Using yq to Manipulate This Composition

### Update Node Count

```bash
yq -i '
  (.spec.resources[] | 
   select(.name == "akscluster").base.spec.forProvider.defaultNodePool[0].nodeCount) = 5
' azure-aks-composition.yaml
```

### Change Location for All Resources

```bash
yq -i '
  (.spec.resources[].base.spec.forProvider.location) = "northeurope"
' azure-aks-composition.yaml
```

### Add New Patch

```bash
yq -i '
  (.spec.resources[] | 
   select(.name == "akscluster").patches) += [{
    "type": "FromCompositeFieldPath",
    "fromFieldPath": "spec.parameters.tags",
    "toFieldPath": "spec.forProvider.tags"
  }]
' azure-aks-composition.yaml
```

### Update Kubernetes Version

```bash
yq -i '
  (.spec.resources[] | 
   select(.name == "akscluster").base.spec.forProvider.kubernetesVersion) = "1.29"
' azure-aks-composition.yaml
```

### Add Resource Group Tags

```bash
yq -i '
  (.spec.resources[] | 
   select(.name == "resourcegroup").base.spec.forProvider.tags) = {
    "environment": "production",
    "managed-by": "crossplane",
    "cost-center": "engineering"
  }
' azure-aks-composition.yaml
```

### Extract All Resource Names

```bash
yq '.spec.resources[].name' azure-aks-composition.yaml
```

### Get Patch Configuration for AKS

```bash
yq '.spec.resources[] | select(.name == "akscluster") | .patches' azure-aks-composition.yaml
```

### Validate Composition Structure

```bash
# Check all resources have names
yq '.spec.resources[] | select(.name == null) | "ERROR: Resource without name"' azure-aks-composition.yaml

# Check all resources have provider config
yq '.spec.resources[] | select(.base.spec.providerConfigRef == null) | .name' azure-aks-composition.yaml

# List all resource kinds
yq '.spec.resources[] | {name: .name, kind: .base.kind}' azure-aks-composition.yaml
```

## yq Commands for Common Tasks

### Environment-Specific Variants

```bash
# Create dev variant
yq '
  .metadata.name = "azure-aks-cluster-dev" |
  .metadata.labels.environment = "dev" |
  (.spec.resources[] | select(.name == "akscluster").base.spec.forProvider.defaultNodePool[0].nodeCount) = 1 |
  (.spec.resources[] | select(.name == "akscluster").base.spec.forProvider.defaultNodePool[0].vmSize) = "Standard_B2s"
' azure-aks-composition.yaml > azure-aks-composition-dev.yaml

# Create prod variant
yq '
  .metadata.name = "azure-aks-cluster-prod" |
  .metadata.labels.environment = "prod" |
  (.spec.resources[] | select(.name == "akscluster").base.spec.forProvider.defaultNodePool[0].nodeCount) = 5 |
  (.spec.resources[] | select(.name == "akscluster").base.spec.forProvider.defaultNodePool[0].vmSize) = "Standard_D4s_v3"
' azure-aks-composition.yaml > azure-aks-composition-prod.yaml
```

### Audit Configuration

```bash
# Check auto-scaling configuration
yq '.spec.resources[] | 
  select(.name == "akscluster") | 
  .base.spec.forProvider.defaultNodePool[0] | 
  {
    enableAutoScaling,
    minCount,
    maxCount,
    nodeCount
  }' azure-aks-composition.yaml

# List all patches by type
yq '.spec.resources[].patches[] | 
  group_by(.type) | 
  map({type: .[0].type, count: length})' azure-aks-composition.yaml
```

## Testing

```bash
# Validate YAML syntax
yq '.' azure-aks-composition.yaml > /dev/null && echo "Valid" || echo "Invalid"

# Check required Crossplane fields
yq '.spec.compositeTypeRef.kind' azure-aks-composition.yaml
yq '.spec.resources | length' azure-aks-composition.yaml

# Verify all resources have provider configs
yq '.spec.resources[] | 
  select(.base.spec.providerConfigRef == null) | 
  "Missing provider config: " + .name' azure-aks-composition.yaml
```

## Related Files

- **XRD (CompositeResourceDefinition)**: Defines the API schema
- **Claim**: User-facing resource that uses this composition
- **Provider Config**: Azure credentials and configuration

This example demonstrates a real-world Crossplane Composition that you might use in your NATO/Atos IDP project for provisioning AKS clusters with yq for manipulation and management.
