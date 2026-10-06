# Klusterlet AgentConfiguration API Proposal

## Context

**Related Issue**: https://github.com/open-cluster-management-io/api/issues/439  
**Related OCM Issue**: https://github.com/open-cluster-management-io/ocm/issues/1032

Currently, the health check port for klusterlet agents is hardcoded to 8443. When using `hostNetwork: true`, this creates port conflicts if 8443 is already in use, causing pod startup failures.

## Problem

The current PR #1688 in ocm repo proposes adding a single `healthCheckPort` field:

```go
type KlusterletSpec struct {
    // ... existing fields ...
    HealthCheckPort int32 `json:"healthCheckPort,omitempty"`
}
```

**Issues with this approach:**
1. ❌ Too narrow - only solves one specific issue
2. ❌ Not extensible - what if we need to configure metrics port, other health parameters, or host network mode?
3. ❌ Inconsistent with ClusterManager API pattern
4. ❌ Doesn't follow OCM conventions for sibling resources

## Proposed Solution: AgentConfiguration

Follow the **ClusterManager's `BindConfiguration` pattern** (see [types_clustermanager.go:424-451](https://github.com/open-cluster-management-io/api/blob/main/operator/v1/types_clustermanager.go#L424-L451))

### API Design

Add to `KlusterletSpec`:

```go
type KlusterletSpec struct {
    // ... existing fields ...
    
    // AgentConfiguration contains the configuration for klusterlet agents.
    // This includes bind configuration for health probes, metrics, and other agent-specific settings.
    // +optional
    AgentConfiguration *AgentConfiguration `json:"agentConfiguration,omitempty"`
}
```

New types:

```go
// AgentConfiguration represents customization of klusterlet agent deployments
type AgentConfiguration struct {
    // BindConfiguration represents server bind configuration for agent health and metrics endpoints.
    // This applies to registration-agent and work-agent deployments.
    // +optional
    BindConfiguration *AgentBindConfiguration `json:"bindConfiguration,omitempty"`
}

// AgentBindConfiguration represents customization of agent server bindings
type AgentBindConfiguration struct {
    // HealthProbePort represents the bind port for the agent's health check endpoint (liveness/readiness probes).
    // The default value is 8443.
    // In hostNetwork mode, ensure this port doesn't conflict with other services on the node.
    // Note: Ports below 1024 require privileged containers (agents run non-root by default).
    // +optional
    // +kubebuilder:default=8443
    // +kubebuilder:validation:Minimum=1024
    // +kubebuilder:validation:Maximum=65535
    HealthProbePort int32 `json:"healthProbePort,omitempty"`
    
    // MetricsPort represents the bind port for the agent's metrics endpoint.
    // The default value is 8080.
    // Metrics may be disabled by setting a value less than or equal to 0.
    // +optional
    // +kubebuilder:default=8080
    // +kubebuilder:validation:Maximum=65535
    MetricsPort int32 `json:"metricsPort,omitempty"`
    
    // HostNetwork enables running agent pods in host networking mode.
    // When enabled, pods use the host's network namespace instead of a pod-specific namespace.
    // This may be required in certain network configurations or when the CNI doesn't support pod networking.
    // Note: When enabled, ensure HealthProbePort and MetricsPort don't conflict with other host services.
    // +optional
    HostNetwork bool `json:"hostNetwork,omitempty"`
}
```

### Example Usage

**Scenario 1: Change health check port for hostNetwork deployment**
```yaml
apiVersion: operator.open-cluster-management.io/v1
kind: Klusterlet
metadata:
  name: klusterlet
spec:
  clusterName: my-cluster
  agentConfiguration:
    bindConfiguration:
      healthProbePort: 9443  # Custom port to avoid conflict
      hostNetwork: true      # Enable host networking
```

**Scenario 2: Separate health and metrics ports**
```yaml
apiVersion: operator.open-cluster-management.io/v1
kind: Klusterlet
metadata:
  name: klusterlet
spec:
  clusterName: my-cluster
  agentConfiguration:
    bindConfiguration:
      healthProbePort: 8443
      metricsPort: 9090     # Custom metrics port
```

**Scenario 3: Default behavior (backward compatible)**
```yaml
apiVersion: operator.open-cluster-management.io/v1
kind: Klusterlet
metadata:
  name: klusterlet
spec:
  clusterName: my-cluster
  # agentConfiguration not specified - uses defaults:
  # healthProbePort: 8443, metricsPort: 8080, hostNetwork: false
```

## Benefits

### ✅ Extensibility
- Easy to add future agent-specific configurations without breaking changes
- Can add more bind configuration options (e.g., `PProfPort`, `DebugPort`)
- Can add non-bind agent configs under `AgentConfiguration` later

### ✅ Consistency with ClusterManager
- Mirrors `ClusterManager.Spec.DeployOption.Default.{Registration,Work,Addon}WebhookConfiguration.BindConfiguration`
- Same field names: `HealthProbePort`, `MetricsPort`, `HostNetwork`
- Developers already familiar with this pattern from hub-side configuration

### ✅ Future-Proof
```go
// Future expansion examples:
type AgentConfiguration struct {
    BindConfiguration *AgentBindConfiguration `json:"bindConfiguration,omitempty"`
    
    // Future additions (non-breaking):
    // LogLevel *string `json:"logLevel,omitempty"`
    // ProbeConfiguration *ProbeConfiguration `json:"probeConfiguration,omitempty"`
    // ResourceOverrides map[string]ResourceRequirement `json:"resourceOverrides,omitempty"`
}
```

### ✅ Backward Compatible
- Optional field (`+optional`)
- Defaults maintain current behavior
- Existing Klusterlet CRs continue to work without modification

## Implementation Plan

### Phase 1: API Changes (api repo)
1. Add `AgentConfiguration` and `AgentBindConfiguration` types to `operator/v1/types_klusterlet.go`
2. Add `AgentConfiguration *AgentConfiguration` field to `KlusterletSpec`
3. Update CRD generation with proper validation tags
4. Add comprehensive godoc comments
5. Update vendored types in dependent repos

### Phase 2: Controller Implementation (ocm repo)
1. Update klusterlet controller to read `spec.agentConfiguration.bindConfiguration`
2. Apply health probe port to agent deployment manifests
3. Apply metrics port to agent deployment manifests  
4. Apply hostNetwork setting to pod specs
5. Maintain backward compatibility (defaults when config not specified)
6. Add validation: reject invalid port ranges, ensure non-privileged ports

### Phase 3: Testing & Documentation
1. Add unit tests for configuration parsing and defaults
2. Add integration tests with various configurations
3. Test hostNetwork + custom port scenarios
4. Update klusterlet operator documentation
5. Add migration guide for existing deployments
6. Update examples in api repo

## Validation Rules

### CRD-Level Validation
- `healthProbePort`: 1024-65535 (non-privileged ports, since agents run non-root)
- `metricsPort`: 0-65535 (0 = disabled)
- Port values use kubebuilder validation tags

### Controller-Level Validation
- When `hostNetwork: true` is set, log warning about port conflict risks
- Reject configuration if `healthProbePort == metricsPort` (same non-zero value)
- Default `0` or unset values to documented defaults before applying

## Comparison with Current PR

| Aspect | Current PR #1688 | This Proposal |
|--------|-----------------|---------------|
| **Field Location** | `KlusterletSpec.HealthCheckPort` | `KlusterletSpec.AgentConfiguration.BindConfiguration.HealthProbePort` |
| **Extensibility** | ❌ Not extensible | ✅ Easy to extend with more agent configs |
| **Consistency** | ❌ Unique to Klusterlet | ✅ Matches ClusterManager pattern |
| **Metrics Port** | ❌ Not configurable | ✅ Included |
| **Host Network** | ❌ Separate field? | ✅ Grouped with bind config |
| **Future Additions** | ❌ Requires new top-level fields | ✅ Add under AgentConfiguration |
| **Backward Compat** | ✅ Yes | ✅ Yes |

## Migration Path

For users who might have adopted the single-field approach from PR #1688 (if it had been merged):

```go
// Migration helper in controller:
func getHealthProbePort(klusterlet *operatorv1.Klusterlet) int32 {
    // Prefer new structured config
    if klusterlet.Spec.AgentConfiguration != nil &&
       klusterlet.Spec.AgentConfiguration.BindConfiguration != nil &&
       klusterlet.Spec.AgentConfiguration.BindConfiguration.HealthProbePort > 0 {
        return klusterlet.Spec.AgentConfiguration.BindConfiguration.HealthProbePort
    }
    
    // Default
    return constants.DefaultAgentHealthProbePort // 8443
}
```

## Open Questions

1. **Should we include separate config for registration-agent vs work-agent?**
   - Current proposal: Single configuration applies to both
   - Alternative: `RegistrationAgentConfiguration` + `WorkAgentConfiguration` for per-agent customization
   - **Recommendation**: Start with single config, split later if needed

2. **Should MetricsPort be in this initial PR?**
   - **Recommendation**: Yes - it's already in ClusterManager's BindConfiguration pattern, adds minimal complexity

3. **Validation: CRD-level vs controller-level?**
   - **Recommendation**: Use both - CRD for simple range checks, controller for complex validation (port conflicts, etc.)

4. **Should hostNetwork be in BindConfiguration or at AgentConfiguration level?**
   - **Recommendation**: Keep in BindConfiguration (matches ClusterManager pattern exactly)

## References

- **ClusterManager BindConfiguration**: [types_clustermanager.go:424-451](https://github.com/open-cluster-management-io/api/blob/main/operator/v1/types_clustermanager.go#L424-L451)
- **ClusterManager DefaultWebhookConfiguration**: [types_clustermanager.go:453-457](https://github.com/open-cluster-management-io/api/blob/main/operator/v1/types_clustermanager.go#L453-L457)
- **Issue #439**: https://github.com/open-cluster-management-io/api/issues/439
- **OCM Issue #1032**: https://github.com/open-cluster-management-io/ocm/issues/1032

## Next Steps

1. **Gather feedback** on this proposal in issue #439
2. **Reach consensus** on the API structure with maintainers
3. **Create PR in api repo** with the types and CRD updates
4. **Wait for api PR merge** and new release
5. **Update ocm repo** to consume new API and implement controller logic
6. **Update PR #1688** to use the new structured configuration
