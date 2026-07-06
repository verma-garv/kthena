---
title: Role-Scoped Network Topology Policies in ModelServing
authors:
- "@LiZhenCheng9527"
reviewers:
- TBD
approvers:
- TBD

creation-date: 2026-07-06

---

## Role-Scoped Network Topology Policies in ModelServing

### Summary

This proposal introduces role-scoped network topology policies for ModelServing while preserving backward compatibility with the existing `spec.template.networkTopology.rolePolicy` API. Today, `rolePolicy` is copied to every generated `PodGroup.spec.subGroupPolicy` entry, so all roles in a ModelServing receive the same network topology constraint. This is too coarse for heterogeneous inference services where different roles have different scheduling requirements.

The proposed transition adds an optional role-level network topology policy to each ModelServing role. When a role defines its own policy, that policy takes precedence. When the role does not define one, the controller falls back to the existing ModelServing-level `rolePolicy`. This allows users to gradually migrate to more precise per-role configuration without breaking existing manifests.

### Motivation

PD-disaggregated inference services commonly contain multiple roles with different resource and scheduling requirements. For example, `prefill` and `decode` roles may require GPUs, RDMA resources, and network-topology-aware scheduling, while an `lb` role may be CPU-only and should run on general-purpose nodes.

The current ModelServing API only supports one `spec.template.networkTopology.rolePolicy` for all roles. When that policy is configured, the generated `PodGroup.spec.subGroupPolicy` for every role receives the same network topology constraint. As a result, roles that do not require network-topology-aware scheduling may be unnecessarily restricted to HyperNodes, may consume scarce RDMA-capable nodes, or may remain pending if general-purpose nodes are outside the required topology tier.

Using `groupPolicy` does not solve this issue because it applies to all pods in the ServingGroup. Splitting roles such as `lb` into a separate workload is also undesirable when those roles are logically and operationally part of the same ModelServing.

#### Goals

1. Allow each ModelServing role to define its own network topology scheduling policy.
2. Preserve backward compatibility with the existing `spec.template.networkTopology.rolePolicy` behavior.
3. Allow roles without a role-level policy and without a fallback `rolePolicy` to omit network topology constraints from their generated `SubGroupPolicy`.
4. Make the precedence between role-level and ModelServing-level network topology policies explicit and predictable.
5. Enable heterogeneous roles, such as `prefill`, `decode`, and `lb`, to remain in one ModelServing while applying topology constraints only where needed.

#### Non-Goals

1. This proposal does not remove `spec.template.networkTopology.rolePolicy` in the initial implementation.
2. This proposal does not change the semantics of `spec.template.networkTopology.groupPolicy`.

### Proposal

Add an optional network topology policy field to each ModelServing role. The controller will use this field when generating the corresponding `PodGroup.spec.subGroupPolicy[*].networkTopology`.

The policy selection rules are:

1. If a role defines a role-level network topology policy, use that role-level policy.
2. If a role does not define a role-level policy and `spec.template.networkTopology.rolePolicy` is set, inherit the ModelServing-level `rolePolicy`.
3. If only the role defines a policy, use the role-level policy.
4. If neither the role nor the ModelServing-level `rolePolicy` defines a policy, do not set `networkTopology` for that role's `SubGroupPolicy`.

This keeps existing manifests working as before because a ModelServing that only configures `spec.template.networkTopology.rolePolicy` will still apply that policy to every role. New manifests can omit the global `rolePolicy` and configure policies only on selected roles.

#### User Stories (Optional)

##### Story 1: Apply topology constraints only to inference roles

A user deploys a PD-disaggregated ModelServing with `prefill`, `decode`, and `lb` roles. The `prefill` and `decode` roles require hard network-topology-aware scheduling with `highestTierAllowed: 1`. The `lb` role is CPU-only and does not require RDMA or topology-aware scheduling.

With role-level policies, the user configures network topology only for `prefill` and `decode`. The generated PodGroup contains topology constraints for those two roles, while the `lb` role's SubGroupPolicy has no `networkTopology` field and can be scheduled onto general-purpose nodes.

##### Story 2: Preserve existing global role policy behavior

An existing user already has ModelServing manifests that configure `spec.template.networkTopology.rolePolicy`. After upgrading the controller, the manifests continue to work without modification. Since the roles do not define role-level policies, each role still inherits the existing ModelServing-level `rolePolicy`.

##### Story 3: Override the default policy for one role

A user wants most roles to use a default ModelServing-level `rolePolicy`, but one role needs a stricter or looser topology policy. The user keeps `spec.template.networkTopology.rolePolicy` as the default and sets a role-level policy only on the exceptional role. The controller uses the role-level policy for that role and the default policy for the remaining roles.

#### Notes/Constraints/Caveats (Optional)

The existing `spec.template.networkTopology.rolePolicy` becomes a compatibility fallback and default policy for roles that do not specify their own policy. It should not be removed as part of this proposal.

This proposal intentionally does not add an explicit opt-out field. If users need one role to avoid inheriting the global `rolePolicy`, they should omit the global `rolePolicy` and configure role-level policies on the roles that need topology constraints. An explicit opt-out can be considered separately if a future use case requires a global default policy with individual exceptions.

The role-level policy applies to the role's generated `SubGroupPolicy`, not directly to individual pods. This follows the current PodGroup integration model.

#### Risks and Mitigations

### Design Details

#### API Changes

Add an optional `NetworkTopology` field to `Role`:

```go
type Role struct {
    // Existing fields omitted.

    // NetworkTopology defines the network topology scheduling requirement for this role.
    // When set, it takes precedence over spec.template.networkTopology.rolePolicy.
    // +optional
    NetworkTopology *volcanoV1Beta1.NetworkTopologySpec `json:"networkTopology,omitempty"`
}
```

The existing `NetworkTopology` type remains unchanged in the compatibility phase:

```go
type NetworkTopology struct {
    // GroupPolicy defines the network topology scheduling requirement of all instances within the ServingGroup.
    GroupPolicy *volcanoV1Beta1.NetworkTopologySpec `json:"groupPolicy,omitempty"`

    // RolePolicy defines the default network topology scheduling requirement for roles.
    // This field is retained for backward compatibility and is used only when a role does not define its own networkTopology.
    RolePolicy *volcanoV1Beta1.NetworkTopologySpec `json:"rolePolicy,omitempty"`
}
```

#### Example

Selected roles define their own topology policies, while `lb` receives no topology constraint:

```yaml
apiVersion: workload.serving.volcano.sh/v1alpha1
kind: ModelServing
metadata:
  name: pd-disaggregated-sample
spec:
  schedulerName: volcano
  template:
    roles:
    - name: prefill
      replicas: 2
      workerReplicas: 1
      networkTopology:
        mode: hard
        highestTierAllowed: 1
      entryTemplate:
        spec:
          containers:
          - name: prefill
            image: example/prefill:latest
    - name: decode
      replicas: 2
      workerReplicas: 1
      networkTopology:
        mode: hard
        highestTierAllowed: 1
      entryTemplate:
        spec:
          containers:
          - name: decode
            image: example/decode:latest
    - name: lb
      replicas: 2
      workerReplicas: 0
      entryTemplate:
        spec:
          containers:
          - name: lb
            image: example/lb:latest
```

The generated `PodGroup.spec.subGroupPolicy` should contain `networkTopology` for `prefill` and `decode`, but not for `lb`:

```yaml
spec:
  subGroupPolicy:
  - labelSelector:
      matchLabels:
        modelserving.volcano.sh/name: sample
        modelserving.volcano.sh/role: prefill
    networkTopology:
      mode: hard
      highestTierAllowed: 1
  - labelSelector:
      matchLabels:
        modelserving.volcano.sh/name: sample
        modelserving.volcano.sh/role: decode
    networkTopology:
      mode: hard
      highestTierAllowed: 1
```

#### Controller Behavior

When building `PodGroup.spec.subGroupPolicy`, the controller should resolve the topology policy for each role independently:

```go
func resolveRoleNetworkTopology(ms *workloadv1alpha1.ModelServing, role workloadv1alpha1.Role) *schedulingv1beta1.NetworkTopologySpec {
    if role.NetworkTopology != nil {
        return role.NetworkTopology
    }
    if ms.Spec.Template.NetworkTopology != nil {
        return ms.Spec.Template.NetworkTopology.RolePolicy
    }
    return nil
}
```

Then `appendSubGroupPolicy` should assign `SubGroupPolicySpec.NetworkTopology` only when the resolved policy is non-nil.

#### Compatibility and Migration

Existing manifests continue to work:

```yaml
spec:
  template:
    networkTopology:
      rolePolicy:
        mode: hard
        highestTierAllowed: 1
    roles:
    - name: prefill
    - name: decode
```

Both roles inherit the existing `rolePolicy` because they do not define role-level policies.

Users who need selective application should migrate by removing the global `rolePolicy` and adding `networkTopology` only to the roles that need it.

#### Test Plan

### Alternatives
