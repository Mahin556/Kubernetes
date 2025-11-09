## **Pod Conditions**
  * While **Pod phase** gives a broad status, **Pod conditions** provide **detailed state information** about scheduling, readiness, and initialization.
  **Common Conditions:**

  | **Condition**       | **Description**                                          |
  | ------------------- | -------------------------------------------------------- |
  | **PodScheduled**    | True if Pod has been successfully scheduled onto a node. |
  | **Ready**           | True if the Pod is ready to serve traffic.               |
  | **Initialized**     | True if all init containers have completed successfully. |
  | **ContainersReady** | True if all containers in the Pod are ready.             |

**Example Command:**

```bash
kubectl describe pod java-api-pod
# Look for the "Conditions" section
```

**Example Output Snippet:**

```
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
```

> **Key Insight:** Conditions are part of the `PodStatus` object and help controllers or users understand **why a Pod is in a particular phase**.