# OpenShift — Scenario Based

---

## 1. How would you design a multi-tenant cluster in OpenShift?

Multi-tenancy means multiple teams share one OpenShift cluster while being isolated from each other — isolation in compute, network, storage, and access. OpenShift is built for this better than vanilla K8s because Projects, SCCs, and network policies are first-class features.

**The four pillars of multi-tenancy:**

```
Multi-Tenant Cluster
├── 1. Project isolation     → separate namespaces per team
├── 2. RBAC isolation        → team A can't see team B's resources
├── 3. Network isolation     → team A's pods can't reach team B's pods
└── 4. Resource isolation    → team A can't consume all cluster resources
```

**Pillar 1 — Project isolation:**

Each team gets their own Project (namespace). Use a project template to enforce standards automatically:

```yaml
# Project template — every new project gets these resources
apiVersion: template.openshift.io/v1
kind: Template
metadata:
  name: project-request
  namespace: openshift-config
objects:
  # Resource quotas — prevent one team from consuming all cluster resources
  - apiVersion: v1
    kind: ResourceQuota
    metadata:
      name: compute-quota
    spec:
      hard:
        requests.cpu: "8"
        requests.memory: 16Gi
        limits.cpu: "16"
        limits.memory: 32Gi
        pods: "50"
        services: "20"
        persistentvolumeclaims: "10"

  # Limit ranges — default container limits (no pod runs without limits)
  - apiVersion: v1
    kind: LimitRange
    metadata:
      name: default-limits
    spec:
      limits:
        - default:
            cpu: 500m
            memory: 512Mi
          defaultRequest:
            cpu: 100m
            memory: 128Mi
          type: Container

  # Network policy — deny all inter-namespace traffic by default
  - apiVersion: networking.k8s.io/v1
    kind: NetworkPolicy
    metadata:
      name: deny-all-ingress
    spec:
      podSelector: {}
      policyTypes: [Ingress]
      ingress: []         # No inbound traffic from other namespaces
```

```bash
# Apply the project template
oc create -f project-template.yaml -n openshift-config
# Update the cluster config to use it
oc edit project.config.openshift.io/cluster
# Set: spec.projectRequestTemplate.name: project-request
```

**Pillar 2 — RBAC isolation:**

Each team gets `admin` role in their project only. No cross-project access:

```bash
# Team A gets admin in their project
oc adm policy add-role-to-group admin team-a -n team-a-project

# Team A has NO access to team B's project (default — no RoleBinding exists)

# Shared services (monitoring, logging) get view access across projects
oc adm policy add-cluster-role-to-user cluster-reader monitoring-sa

# Disable self-provisioning (prevent users from creating their own projects)
oc adm policy remove-cluster-role-from-group self-provisioner system:authenticated:oauth
# Now only cluster-admins can create projects → controlled onboarding
```

**Pillar 3 — Network isolation:**

OpenShift's SDN (OVN-Kubernetes or OpenShift SDN) supports network policies natively:

```yaml
# Allow traffic only within the same namespace + from ingress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-same-namespace
  namespace: team-a-project
spec:
  podSelector: {}
  policyTypes: [Ingress]
  ingress:
    - from:
        - podSelector: {}          # Same namespace — allowed
    - from:
        - namespaceSelector:
            matchLabels:
              network.openshift.io/policy-group: ingress   # Router — allowed
```

This means team A's pods can communicate with each other, the router can reach them (for external traffic), but team B's pods cannot reach team A's pods at all.

**Pillar 4 — Resource isolation:**

ResourceQuotas prevent resource hogging. ClusterResourceQuotas can span multiple projects for a team:

```yaml
# ClusterResourceQuota — limits total resources across ALL of team A's projects
apiVersion: quota.openshift.io/v1
kind: ClusterResourceQuota
metadata:
  name: team-a-quota
spec:
  quota:
    hard:
      requests.cpu: "20"
      requests.memory: 40Gi
  selector:
    labels:
      matchLabels:
        team: team-a           # Applies to all projects with label team=team-a
```

**Node isolation (optional — for strict compliance):**

If teams need dedicated nodes (PCI compliance, GPU workloads):

```bash
# Label nodes for a specific team
oc label node worker-3 team=team-a
oc label node worker-4 team=team-a

# Taint the nodes — only team A's pods can schedule here
oc adm taint nodes worker-3 worker-4 team=team-a:NoSchedule
```

```yaml
# Team A's deployment — tolerate the taint + use nodeSelector
spec:
  nodeSelector:
    team: team-a
  tolerations:
    - key: team
      value: team-a
      effect: NoSchedule
```

**Quick checklist for multi-tenant OpenShift:**

```
☐ Project template with ResourceQuota + LimitRange + NetworkPolicy
☐ RBAC: team admin per project, no cross-project access
☐ Disable self-provisioning (controlled project creation)
☐ NetworkPolicy: deny cross-namespace by default
☐ SCCs: restricted SCC for all tenants (no root, no privilege)
☐ ClusterResourceQuota per team (across all their projects)
☐ Node taints/tolerations for strict isolation (optional)
☐ Separate image streams per project (no shared writable registries)
```

**Real scenario:** We ran an OpenShift 4 cluster shared by 6 teams. Initially, no quotas were set. One team ran a load test that consumed all cluster CPU — every other team's pods were throttled. After implementing the multi-tenancy pattern above (project template with quotas, network policies, RBAC), each team had guaranteed resources. The load-testing team's pods hit their quota ceiling and were throttled instead — everyone else was unaffected. The project template also caught a team that was creating PVCs without limits — they had 200 PVCs consuming 2TB of storage. The quota cap at 10 PVCs per project forced them to clean up unused volumes.

