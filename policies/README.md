# Supplemental ACM Policies

This directory contains supplemental ACM policies that extend or customize the autoshift deployment beyond what's provided by the upstream autoshift Helm charts.

## Policies

### policy-acs-infra-nodes.yaml

**Purpose:** Configures Advanced Cluster Security (ACS) Central to run on infrastructure nodes instead of worker nodes.

**How it works:**
- Uses ACM ConfigurationPolicy with `complianceType: musthave` for additive merging
- Sets explicit `nodeSelector` and `tolerations` in `spec.customize.deploymentDefaults`
- Uses `operator: Exists` toleration to match infra node taints without values
- Sync wave "2" ensures it applies AFTER autoshift's policy-acs-central (wave "1")
- Depends on `policy-acs-central` being compliant before applying

**Note:** We use explicit tolerations instead of `pinToNodes: InfraRole` because that feature assumes infra taints have `value: reserved`, but this automation creates infra nodes with taints that have no value.

**Why this is needed:**
- The upstream autoshift Helm chart doesn't support configuring ACS node placement via cluster labels
- ACS Central has high resource requirements (4 CPU, 8Gi memory limits) that may not fit on smaller worker nodes
- Infrastructure nodes (m5.2xlarge) have ample capacity for ACS components

**Deployment:**
- Automatically applied by `playbooks/deploy-autoshift-stack.yaml` in Step 5
- Targets clusters with label `autoshift.io/acs: 'true'`
- Uses the same namespace (`policies-autoshift`) as autoshift policies

**Compatibility:**
- Works alongside autoshift policies without conflicts
- ArgoCD will not override this policy because it's managed separately
- Safe to run on existing clusters (will patch existing Central CR)

## Adding New Supplemental Policies

When creating new supplemental policies:

1. Use `namespace: policies-autoshift` to keep policies organized
2. Set appropriate `argocd.argoproj.io/sync-wave` (higher than autoshift's wave "1")
3. Add dependencies on upstream autoshift policies if needed
4. Use `complianceType: musthave` for additive merging (avoid conflicts)
5. Add the policy to `playbooks/deploy-autoshift-stack.yaml` Step 5
6. Document the policy here

## Future Considerations

Consider contributing these policies upstream to autoshiftv2:
- File GitHub issues for missing configuration options
- Submit PRs to add cluster labels for common customizations
- This reduces maintenance burden and benefits the community
