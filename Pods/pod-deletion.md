## **1. Why Delete Pods from a Node?**
Deleting pods from a node is commonly needed for:
* **Troubleshooting:** Free up a node to test scheduling or resource issues.
* **Maintenance:** Clear a node before upgrades or repairs.
* **Manual scaling:** Redistribute workloads to other nodes.


## **4. Handle Pod Disruption Budgets (PDB)**
* Error: `Cannot evict pod as it would violate the pod’s disruption budget`.
* PDB ensures minimum availability of pods during voluntary disruptions.
  ```bash
  # List all PDBs
  kubectl get poddisruptionbudget -A

  # Delete a PDB if necessary
  kubectl delete poddisruptionbudget <pdb-name>
  ```
---

## **Key Points**
* **`kubectl delete pod`** removes pods but may trigger recreation if managed by a Deployment or StatefulSet.
* **`kubectl drain`** is safer for nodes—evicts pods while preserving service availability.
* Consider **Pod Disruption Budgets** and **scaling strategies** to prevent downtime.
* Force deletion is a last resort, primarily for stuck pods.

