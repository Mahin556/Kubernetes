
Two type of api-groups:-
  1. core api group
  2. named api group

# **Core (v1) API Objects**

| Object Type           | apiVersion | Notes                            |
| --------------------- | ---------- | -------------------------------- |
| Pod                   | v1         | Smallest deployable unit         |
| Service               | v1         | Exposes Pods to network          |
| ConfigMap             | v1         | Non-sensitive configuration data |
| Secret                | v1         | Sensitive data (passwords, keys) |
| Namespace             | v1         | Logical partitioning of cluster  |
| PersistentVolume      | v1         | Cluster-wide storage resource    |
| PersistentVolumeClaim | v1         | Request for PV                   |
| Node                  | v1         | Cluster node details             |
| Endpoint              | v1         | Endpoints for Services           |
| ServiceAccount        | v1         | Accounts used by Pods            |

---

# **Apps API Group (apps/v1)**

| Object Type | apiVersion | Notes                                        |
| ----------- | ---------- | -------------------------------------------- |
| Deployment  | apps/v1    | Declarative management of Pods + ReplicaSets |
| StatefulSet | apps/v1    | Stateful workloads with stable identity      |
| ReplicaSet  | apps/v1    | Ensures desired number of Pod replicas       |
| DaemonSet   | apps/v1    | One Pod per node for logging/monitoring      |

---

# **Batch API Group (batch/v1)**

| Object Type | apiVersion | Notes                  |
| ----------- | ---------- | ---------------------- |
| Job         | batch/v1   | Run Pods to completion |
| CronJob     | batch/v1   | Scheduled Jobs         |

---

# **Networking API Group**

| Object Type   | apiVersion           | Notes                         |
| ------------- | -------------------- | ----------------------------- |
| Ingress       | networking.k8s.io/v1 | HTTP/HTTPS routing            |
| NetworkPolicy | networking.k8s.io/v1 | Controls traffic between Pods |

---

# **Policy API Group**

| Object Type         | apiVersion | Notes                                            |
| ------------------- | ---------- | ------------------------------------------------ |
| PodDisruptionBudget | policy/v1  | Ensure minimum available Pods during maintenance |

---

# **RBAC API Group**

| Object Type        | apiVersion                   | Notes                       |
| ------------------ | ---------------------------- | --------------------------- |
| Role               | rbac.authorization.k8s.io/v1 | Namespace-level permissions |
| ClusterRole        | rbac.authorization.k8s.io/v1 | Cluster-wide permissions    |
| RoleBinding        | rbac.authorization.k8s.io/v1 | Bind Roles to users/groups  |
| ClusterRoleBinding | rbac.authorization.k8s.io/v1 | Cluster-wide binding        |

---

# **Storage API Group**

| Object Type  | apiVersion        | Notes                         |
| ------------ | ----------------- | ----------------------------- |
| StorageClass | storage.k8s.io/v1 | Storage provisioning policies |

---

# **Custom Resources**

| Object Type                    | apiVersion              | Notes                       |
| ------------------------------ | ----------------------- | --------------------------- |
| CRD (CustomResourceDefinition) | apiextensions.k8s.io/v1 | Define your own objects     |
| Custom Resources               | <group>/<version>       | Managed like native objects |
