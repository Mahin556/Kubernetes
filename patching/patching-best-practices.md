### References:
- https://spacelift.io/blog/kubectl-patch-command

---

### **Tips and Best Practices for Using `kubectl patch`**

1. **Always validate the resource path**

   * `kubectl patch` requires the correct path to the field you want to update.
   * Before running the patch, inspect the resource with:

     ```bash
     kubectl get <resource> <name> -o yaml
     ```
   * This helps ensure you are targeting the correct field and prevents accidental changes.

2. **Choose the right patching strategy**

   * **Strategic Merge Patch (default)**

     * Ideal for updating Pod template fields, labels, annotations, and partial updates to lists.
     * Recognizes Kubernetes-specific structures, such as container names, for intelligent merging.
   * **JSON Merge Patch**

     * Best for overwriting a field or entire sections.
     * Can remove optional fields by setting them to `null`.
     * Useful when you want to fully replace part of a resource, like a container spec.
   * **JSON Patch (RFC 6902)**

     * Provides fine-grained control using explicit operations: `add`, `remove`, `replace`, `move`, `copy`, and `test`.
     * Perfect for modifying items in a list without affecting the entire list, or performing conditional updates.

3. **Dry-run patch commands before execution**

   * Use `--dry-run=client` to preview changes before applying them:

     ```bash
     kubectl patch deployment apache-deployment \
       --type=json \
       --patch '[{"op": "replace", "path": "/spec/template/spec/containers/0/image", "value": "httpd:2.4.62"}]' \
       --dry-run=client -o yaml
     ```
   * This ensures the patch behaves as expected without impacting the live cluster.

4. **Validate resources after patching**

   * Always inspect the updated resource to confirm the patch applied correctly and no unintended fields were changed:

     ```bash
     kubectl get deployment apache-deployment -o yaml
     ```

5. **Avoid patching immutable fields**

   * Certain fields (e.g., `spec.selector` in a Deployment) cannot be modified using `kubectl patch`.
   * To update these fields, export, delete, and reapply the resource:

     ```bash
     kubectl get deployment apache-deployment -o yaml > deployment.yaml
     kubectl delete deployment apache-deployment
     kubectl apply -f deployment.yaml
     ```

6. **Document all changes**

   * Changes made with `kubectl patch` are ad-hoc and can be forgotten.
   * Always update:

     * The original YAML manifest, or
     * A shared knowledge base for your team.
   * Proper documentation prevents inconsistencies in production or future deployments.

7. **Prefer `kubectl apply` in version-controlled environments**

   * In GitOps or CI/CD workflows, maintain YAML manifests as the single source of truth.
   * Use `kubectl apply` for reproducibility and consistent deployments.
   * Use `kubectl patch` only for temporary or ad-hoc changes, and remember to update the manifest to match the live resource if the change should persist.

