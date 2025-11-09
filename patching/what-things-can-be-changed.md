### References:
- https://spacelift.io/blog/kubectl-patch-command

---

### **Common Patchable Fields**

These fields can usually be patched using **any patch type**:

* **Metadata**

  * `labels` ✅
  * `annotations` ✅
  * `finalizers` ✅
  * `ownerReferences` ❌ (immutable after creation for most resources)
* **Spec**

  * Pod template fields in Deployments/ReplicaSets/DaemonSets/StatefulSets:

    * `spec.template.spec.containers[*].image` ✅
    * `spec.template.spec.containers[*].env` ✅
    * `spec.template.spec.containers[*].resources` ✅
    * `spec.template.spec.containers[*].ports` ✅
    * `spec.template.spec.tolerations` ✅
    * `spec.template.spec.nodeSelector` ✅
    * `spec.template.spec.affinity` ✅
    * `spec.replicas` ✅ (for Deployments, StatefulSets, ReplicaSets)
  * **ServiceSpec**:

    * `ports` ✅ (add/update)
    * `selector` ❌ (immutable for Services)
  * **ConfigMaps / Secrets**:

    * `data` ✅
    * `binaryData` ✅

---

### **2. Common Immutable Fields (Cannot Patch)**

These fields are **frozen after creation** and require deleting/recreating the resource if you want to change them:

* `metadata.name` ❌
* `metadata.namespace` ❌
* `spec.selector` ❌ (for Deployments, Services, ReplicaSets)
* `spec.clusterIP` ❌ (for Services)
* `spec.type` ❌ (for Services, mostly)
* `spec.volumeClaimTemplates` ❌ (for StatefulSets)
* `spec.persistentVolumeClaim.spec.storageClassName` ❌ (PVC after creation)
* `spec.persistentVolumeClaim.spec.accessModes` ❌ (PVC)
* `status` fields ❌ (usually updated by controllers only, not patchable manually)

---

### **3. Rules for Patchable Fields**

* **Strategic Merge Patch**: Works on fields recognized by Kubernetes, especially **lists with unique keys**, e.g., containers (`name` is key).
* **JSON Merge Patch**: Overwrites entire fields; good for replacing objects, but careful with lists (can remove items unintentionally).
* **JSON Patch**: Can patch any field, even inside lists, as long as you know the exact path.

---

### **4. How to Check if a Field is Patchable**

1. Inspect the resource YAML:

   ```bash
   kubectl get deployment my-deployment -o yaml
   ```
2. Check the Kubernetes API documentation for **immutable fields**.
3. Attempt a `--dry-run=client` patch:

   ```bash
   kubectl patch deployment my-deployment \
     --type=json \
     --patch '[{"op": "replace", "path": "/spec/immutableField", "value": "new"}]' \
     --dry-run=client
   ```

   * If the field is immutable, the dry-run will fail with a clear error.
