* Kubectl utility version can be only one minor version difference of cluster.
* For example, a v1.34 client can communicate with v1.33, v1.34, and v1.35 control planes. Using the latest compatible version of kubectl helps avoid unforeseen issues.

### [Using CURL on Linux](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/#install-kubectl-binary-with-curl-on-linux)

* Latest version(release)
```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl" #x86-64
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/arm64/kubectl" #ARM64
```

* Specific version
```bash
curl -LO https://dl.k8s.io/release/v1.34.0/bin/linux/amd64/kubectl #x86-64
curl -LO https://dl.k8s.io/release/v1.34.0/bin/linux/arm64/kubectl #ARM64
```

* Install kubectl
```bash
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```
Note:
If you do not have root access on the target system, you can still install kubectl to the ~/.local/bin directory:
```bash
chmod +x kubectl
mkdir -p ~/.local/bin
mv ./kubectl ~/.local/bin/kubectl
# and then append (or prepend) ~/.local/bin to $PATH
```

* Test to ensure the version you installed is up-to-date:
```bash
kubectl version --client
kubectl version --client --output=yaml #detailed view
```


### [Install using native package management](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/#install-using-native-package-management)

### [Enable shell autocompletion](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/#enable-shell-autocompletion)

### References:
- https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/