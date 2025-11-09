
### Create YAML Manifest Using Kubectl Dry Run
#### [Dry-run flag](https://github.com/Mahin556/K8S-artifects/blob/main/Cluster/dry-run.md) <-- see here

#### Generate Pod YAML (client-side dry-run)
```bash
kubectl run mypod \
  --image=nginx:1.25 \
  --restart=Never \
  --port=80 \
  --labels="app=nginx,tier=frontend" \
  --annotations="description='test pod'" \
  --dry-run=client -o yaml > pod.yaml
```
* Server side validation
```bash
kubectl apply -f pod.yaml --dry-run=server
```

#### Generate Deployment YAML (client-side dry-run)
```bash
kubectl create deployment mydeployment \
  --image=nginx:1.25 \
  --replicas=3 \
  --dry-run=client -o yaml > deployment.yaml
```
* Server side validation
```bash
kubectl apply -f deployment.yaml --dry-run=server
```

#### Generate ClusterIP Service YAML
```bash
kubectl expose deployment mydeployment \
  --port=80 --target-port=8080 \
  --name=myservice \
  --type=ClusterIP \
  --dry-run=client -o yaml > service.yaml
```
* Create Deployment Service YAML
```bash
kubectl expose deployment mydeployment \
    --type=NodePort \
    --port=8080 \
    --name=mydeployment-service \
    --dry-run=client -o yaml > mydeployment-service.yaml
```
* Create a Pod service YAML
```bash
kubectl expose pod mypod \
    --port=80 \
    --name mypod-service \
    --type=NodePort \
    --dry-run=client -o yaml > mypod-service.yaml
```
* Create NodePort Service YAML
```bash
kubectl create service nodeport mypod \
    --tcp=80:80 \
    --node-port=30001 \
    --dry-run=client -o yaml > mypod-service.yaml
```
* Server side validation
```bash
kubectl apply -f service.yaml --dry-run=server
```

#### Generate ConfigMap YAML
```bash
kubectl create configmap myconfig \
  --from-literal=key1=value1 \
  --from-literal=key2=value2 \
  --dry-run=client -o yaml > configmap.yaml
```
* Server side validation
```bash
kubectl apply -f configmap.yaml --dry-run=server
```

#### Generate Secret YAML
```bash
kubectl create secret generic mysecret \
  --from-literal=username=admin \
  --from-literal=password=1234 \
  --dry-run=client -o yaml > secret.yaml
```
* Server side validation
```bash
kubectl apply -f secret.yaml --dry-run=server
```

#### Generate Ingress YAML
```bash
kubectl create ingress myingress \
  --rule="host.example.com/*=myservice:80" \
  --dry-run=client -o yaml > ingress.yaml
```
* Server side validation
```bash
kubectl apply -f ingress.yaml --dry-run=server
```

#### Generate Namespace YAML
```bash
kubectl create namespace mynamespace \
  --dry-run=client -o yaml > namespace.yaml
```
* Server side validation
```bash
kubectl apply -f namespace.yaml --dry-run=server
```

### Create Job YAML
```bash
kubectl create job myjob \
    --image=nginx:latest \
    --dry-run=client -o yaml
```

### Create Cronjob YAML
```bash
kubectl create cj mycronjob \
    --image=nginx:latest \
    --schedule="* * * * *" \
    --dry-run=client -o yaml
```



#### Single imperative command to generate a Pod YAML manifest with everything
```bash
kubectl run mypod \
  --image=nginx:1.25 \
  --port=80 \
  --image-pull-policy=Always \
  --restart=Never \
  --overrides='
  {
    "apiVersion": "v1",
    "metadata": {
      "labels": {
        "app": "nginx",
        "tier": "frontend"
      },
      "annotations": {
        "description": "Sample nginx pod"
      },
      "namespace": "mynamespace"
    },
    "spec": {
      "containers": [
        {
          "name": "nginx-container",
          "image": "nginx:1.25",
          "ports": [
            {
              "containerPort": 80
            }
          ],
          "imagePullPolicy": "Always"
        }
      ],
      "restartPolicy": "Never"
    }
  }' \
  --dry-run=client -o yaml > pod.yaml
```
Resulting YAML (pod.yaml)
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
  namespace: mynamespace
  labels:
    app: nginx
    tier: frontend
  annotations:
    description: Sample nginx pod
spec:
  restartPolicy: Never
  containers:
  - name: nginx-container
    image: nginx:1.25
    imagePullPolicy: Always
    ports:
    - containerPort: 80

```
* --image=nginx:1.25 → container image
* --port=80 → expose port 80 in the container
* --image-pull-policy=Always → always pull image
* --restart=Never → sets pod restart policy
* --overrides → injects extra fields (labels, annotations, namespace, container name)
* --dry-run=client -o yaml → generate YAML instead of creating the resource
* > pod.yaml → save YAML to file


#### Complete imperative command to generate a Deployment YAML
* we can not generate a complete YAML like pod using `kubectl create` because it not support `--overrides` flag.
```bash
kubectl create deployment mydeployment \
  --image=nginx:1.25 \
  --replicas=3 \
  --dry-run=client -o yaml > deployment.yaml
```
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mydeployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: mydeployment
  template:
    metadata:
      labels:
        app: mydeployment
    spec:
      containers:
      - name: mydeployment
        image: nginx:1.25
        ports: []
```

* To add namespace, labels, annotations, custom container name, pull policy, you need to edit the generated YAML manually:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mydeployment
  namespace: mynamespace
  labels:
    app: nginx
    tier: frontend
  annotations:
    description: Sample nginx deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
        tier: frontend
      annotations:
        description: Sample nginx pod template
    spec:
      containers:
      - name: nginx-container
        image: nginx:1.25
        imagePullPolicy: Always
        ports:
        - containerPort: 80
```

```bash
kubectl get pod mypod -o yaml > pod.yaml

kubectl get deployment myapp -o yaml > deployment.yaml

# Remove unnecessory fields from the deployment manifest
kubectl get deployment myapp -o yaml \
  | yq eval 'del(.metadata.uid, .metadata.creationTimestamp, .metadata.resourceVersion, .status)' - \
  > deployment-clean.yaml
```
### Generating YAML Manifest in VS Code
- [Use Kubernetes Extension](https://code.visualstudio.com/docs/azure/kubernetes?ref=devopscube.com)
- Using it we can generate manifest for most of the kubernetes objects:
  - Pod
  - Deployment
  - StatefulSets 
  - ReplicationSet
  - Persistent Volumes(PVs)
  - Persistent Volume claims(PVCs), etc.

- All you have to do is, start typing the Object name and it will automatically populate the options for you. 

![](https://github.com/Mahin556/K8S-artifects/blob/main/images/ks-extension-1.gif)

### References:
- https://devopscube.com/create-kubernetes-yaml/
