### References:-
- [Day 26: Kubernetes Volumes | Ephemeral Storage | emptyDir & downwardAPI DEMO](https://www.youtube.com/watch?v=Zyublb8bSbU&ab_channel=CloudWithVarJosh)
- [Kubernetes - Downward API](https://kubernetes.io/docs/tasks/inject-data-application/downward-api-volume-expose-pod-information/)
- - [Kubernetes - Projected Volumes](https://kubernetes.io/docs/concepts/storage/volumes/#projected)

---

- Enable containers within a Pod to dynamically access runtime metadata of Pod.
- Downward API provides metadata through environment variables and mounted files.

![Alt text](/images/26a.png)

* The Downward API in Kubernetes provides a mechanism for Pods to access metadata about themselves or their environment. This metadata can include information such as the Pod's name, namespace, labels, annotations, or resource limits. It is injected into containers in a Pod either as **environment variables** or through mounted **files via volumes**.

* When the Downward API is configured in a Deployment, each Pod created by the Deployment gets its own unique set of metadata based on the Pod's attributes. This allows Pods to retrieve runtime-specific details dynamically, without hardcoding or manual intervention.

* **Why is it Used?**  
    - **Dynamic Configuration**: Enables applications to dynamically retrieve Pod-specific metadata, such as labels or resource limits.  
    - **Self-Awareness**: Makes Pods aware of their environment, including their name, namespace, and resource constraints.  
    - **Simplifies Configuration Management**: Helps eliminate the need for manual configuration by providing metadata directly to the containers.

* **Sidecar (helper) containers** frequently rely on the **Downward API** to access real-time metadata such as the Pod name, namespace, labels, and resource limits.
    * Without the Downward API, these sidecars would need to **continuously poll the Kubernetes API server** to fetch this metadata, increasing API server load and introducing unnecessary network overhead. By using the Downward API, they can access this data **locally within the Pod**, improving performance and **offloading the API server**.
    * For example, imagine you're running a monitoring agent as a sidecar, and you want to collect metrics or logs for all Pods within a specific namespace like `app1-ns`. If the agent doesn’t know **which Pod it's running in** or **which namespace it belongs to**, it wouldn't be able to label or filter that data correctly. The Downward API solves this problem by **injecting runtime-specific metadata directly into the container**, making it **self-aware**.

<br>

* **Demo: downwardAPI - Environment variables**

    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
    name: downwardapi-example
    labels:
        app: demo
    spec:
    containers:
    - name: metadata-container
        image: busybox
        command: ["/bin/sh", "-c", "env && sleep 3600"] 
        # The container prints all environment variables and then sleeps for 1 hour.
        env:
        - name: POD_NAME
        valueFrom:
            fieldRef:
            fieldPath: metadata.name
        # Creates an environment variable named POD_NAME.
        # The value of this variable is set dynamically using the Downward API.
        # It pulls the Pod's name (in this case, "downwardapi-example") from its metadata.
        - name: POD_NAMESPACE
        valueFrom:
            fieldRef:
            fieldPath: metadata.namespace
        # Creates an environment variable named POD_NAMESPACE.
        # The value of this variable is set dynamically using the Downward API.
        # It pulls the Pod's namespace (e.g., "default") from its metadata.
    ```

    ```bash
    kubectl exec downwardapi-example -- env | grep POD_
    ```

* You should see output like:
    ```bash
    POD_NAME=downwardapi-example
    POD_NAMESPACE=default #take from the namespace section which is auto field when not provided
    ```
    This confirms that:
    - `POD_NAME` is set to the Pod’s name.
    - `POD_NAMESPACE` is set to the namespace it's running in (usually `default`, unless specified otherwise).

<br>

* **Demo: downwardAPI - Files via Volumes**

    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
    name: downwardapi-volume
      labels:
        app: good_app
        owner: hr
      annotations:
        version: "good_version"
    spec:
    containers:
    - name: metadata-container
        image: busybox
        command: ["/bin/sh", "-c", "cat /etc/podinfo/* && sleep 3600"]
        # The container will display the contents of all files under /etc/podinfo (i.e., metadata)
        # and then sleep for an hour, keeping the pod alive for verification.
        volumeMounts:
        - name: downwardapi-volume
          mountPath: /etc/podinfo
        # Mounts the downward API volume at /etc/podinfo inside the container.
    volumes:
    - name: downwardapi-volume
        downwardAPI:
        items:
        - path: "labels"
            fieldRef:
            fieldPath: metadata.labels
            # Writes the Pod's labels to a file named 'labels' under /etc/podinfo.
        - path: "annotations"
            fieldRef:
            fieldPath: metadata.annotations
            # Writes the Pod's annotations to a file named 'annotations' under /etc/podinfo.

    ```
    ```bash
    kubectl exec -it downwardapi-volume -- /bin/sh #When you exec by default it would take you to the first container
    ```
    ```bash
    cd /etc/podinfo
    ls -l
    ```
    You should see symbolic links:
    ```bash
    annotations -> ..data/annotations
    labels -> ..data/labels
    ```
    * These links are created because Kubernetes uses a [projected volume](https://kubernetes.io/docs/concepts/storage/volumes/#projected) with the Downward API, which manages file updates using symlinks pointing to the `..data/` directory. This allows for atomic updates.
    * Check the content of each file:
        ```bash
        cat labels
        ```
    * Expected output:
        ```bash
        app="good_app"
        owner="hr"
        ```

        ```bash
        cat annotations
        ```
    * Expected output:
        ```bash
        version="good_version"
        ```
    * These values are directly fetched from the pod’s metadata and written to files using the Downward API.
    * Since the pod's container command was set to:
        ```yaml
        command: ["/bin/sh", "-c", "cat /etc/podinfo/* && sleep 3600"]
        ```
    * The contents of `/etc/podinfo/labels` and `/etc/podinfo/annotations` will be printed in the pod's logs when it starts. To view them:
        ```bash
        kubectl logs downwardapi-volume
        ```
    * Expected output:
        ```bash
        app="good_app"
        owner="hr"
        version="good_version"
        ```
    * This further confirms that the Downward API volume successfully mounted the metadata into the container at runtime.
