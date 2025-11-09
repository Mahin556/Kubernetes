* Nginx web server pod that shows the Pod IP on the index page.

* Here is how we can utilize init containers to deploy the pod displaying its IP address.
  - One init container named `write-ip` gets the pod IP using the `MY_POD_IP` env variable populated from the Pod's own status. and writes to an `ip.txt` file inside the `/web-content` volume attached to the pod.
  - The second init container named `create-html` reads the pod IP from `/web-content/ip.txt` file that contains the pod IP created by the first init container and writes it to `/web-content/index.html` file.
  - Now, the main nginx container `web-container` mounts the default `/usr/share/nginx/html` to `/web-content` volume where we have the `index.html` file.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-server-pod
spec:
  initContainers:
  - name: write-ip
    image: busybox
    command: ["sh", "-c", "sleep 3; echo Your ip is ${MY_POD_IP:-NotSet} | tee web-content/ip.txt; echo 'Wrote the Pod IP to ip.txt'"]
    env:
    - name: MY_POD_IP
      valueFrom:
        fieldRef:
          fieldPath: status.podIP
    volumeMounts:
    - name: web-content
      mountPath: /web-content
  - name: create-html
    image: busybox
    command: ["sh", "-c", "echo 'Hello, World! Your Pod IP is: ' > /web-content/index.html; cat /web-content/ip.txt >> /web-content/index.html; echo 'Created index.html with the Pod IP'"]
    volumeMounts:
    - name: web-content
      mountPath: /web-content
  containers:
  - name: web-container
    image: nginx
    volumeMounts:
    - name: web-content
      mountPath: /usr/share/nginx/html
  volumes:
  - name: web-content
    emptyDir: {}
```
```bash
kubectl apply -f init-container.yaml
```
```bash
kubectl get pods

NAME             READY   STATUS    RESTARTS   AGE
web-server-pod   1/1     Running   0          22s
```
- There are 3 container in pod but only 1 in running state because init container run to completion.
- Both 2 nginx init container after performing there task they exit with non-zero exit code and then main container starts.

- InitContainers  are exited but we can check there logs.
```bash
$ kubectl logs web-server-pod -c write-ip   
Wrote the Pod IP to ip.txt

$ kubectl logs web-server-pod -c create-html
Created index.html with the Pod IP
```
```bash
kubectl port-forward pod/web-server-pod 8080:80
kubectl port-forward pod/web-server-pod --address 0.0.0.0 8080:80
```
```bash
kill -9 %1
```

### References:
- https://devopscube.com/kubernetes-init-containers/