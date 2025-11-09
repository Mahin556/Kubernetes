```bash
kubectl explain pod.spec.initContainers
```

### Comprehensive Init Container YAML

```yaml
spec:
  initContainers:
    - name: init-container
      image: busybox:latest
      command: ["sh", "-c", "echo Initializing... && sleep 5"]
      imagePullPolicy: IfNotPresent
      restartPolicy: Always #Native sidecar container
      env:
        - name: INIT_ENV_VAR
          value: "init-value"
      resources:
        limits:
          memory: "128Mi"
          cpu: "500m"
        requests:
          memory: "64Mi"
          cpu: "250m"
      volumeMounts:
        - name: init-container-volume
          mountPath: /init-data
      securityContext:
        runAsUser: 1000
        runAsGroup: 1000
        capabilities:
          add: ["NET_ADMIN"]

  containers:
    - name: main-container
      image: nginx:latest
      ports:
        - containerPort: 80
      volumeMounts:
        - name: init-container-volume
          mountPath: /usr/share/nginx/html

  restartPolicy: Always

  volumes:
    - name: init-container-volume
      emptyDir: {}

```

* **Key Points About This YAML**

  1. **Command & Image**

    * `command` runs the initialization logic.
    * `imagePullPolicy: IfNotPresent` avoids redundant image pulls.

  2. **Environment Variables**

    * Used to pass configuration or secrets (if needed).

  3. **Resources**

    * `requests`: Minimum guaranteed CPU/Memory.
    * `limits`: Maximum allowed CPU/Memory.
    * Effective init container resources are the highest of all init containers.

  4. **VolumeMounts & Volumes**

    * Allows sharing data between init containers and main containers.

  6. **Lifecycle Hooks**
    * `postStart`: Commands after container starts.
    * `preStop`: Commands before container stops.

  7. **Security Context**
    * Set user, group, and capabilities to secure the container.

  8. **RestartPolicy**
    * `Always` makes it a **native sidecar** if needed.

  9. **Additional Fields**
    * You can also use: `workingDir`, `volumeDevices`, `resizePolicy` for advanced volume and file system management.

### References:
- https://devopscube.com/kubernetes-init-containers/