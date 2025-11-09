* Your app needs a **secret key** (API key, DB password, token, etc.).
* You cannot **hardcode it in the app image**.
* You cannot use **Kubernetes Secrets directly** (maybe due to compliance/security policies).
* Instead, you want to **fetch secrets at runtime** from a **secret manager** (e.g., HashiCorp Vault, AWS Secrets Manager, Azure Key Vault, GCP Secret Manager).


### 🔹 Solution: Use an Init Container

* The init container:
  1. Connects to the secret manager (with credentials or IAM role).
  2. Fetches the secret.
  3. Writes it to a **shared volume** (e.g., `emptyDir`).
* The main app container:
  * Reads the secret from that volume when it starts.

---

### 🔹 Examples: Of fetching Secrets from different Secret managers

##### 🔹 How This Works

1. **Init Container (`fetch-secret`)**

   * Logs into Vault using a token stored in a Kubernetes Secret (`vault-token`).
   * Fetches the password from Vault path `secret/myapp`.
   * Writes it into `/secret/password.txt` (inside the shared `emptyDir` volume).

2. **Main Container (`myapp`)**

   * Mounts the same shared volume at `/app/secret`.
   * Reads the secret from `password.txt`.

---

![](/images/image-73-9.png)

---

#### 🔹 1. Init Container fetching a secret (Generic Pattern)
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  volumes:
  - name: secret-volume
    emptyDir: { medium: "Memory" }   # shared between init and app containers

  initContainers:
  - name: fetch-secret
    image: curlimages/curl:latest   # lightweight curl image
    command:
      - sh
      - -c
      - |
        echo "Fetching secret..."
        curl -s http://vault.example.com/v1/secret/data/api-key \
          -H "X-Vault-Token: $VAULT_TOKEN" \
          | jq -r '.data.data.key' > /secret/api-key
    env:
    - name: VAULT_TOKEN
      valueFrom:
        secretKeyRef:
          name: vault-auth
          key: token
    volumeMounts:
    - name: secret-volume
      mountPath: /secret

  containers:
  - name: app
    image: nginx:latest
    volumeMounts:
    - name: secret-volume
      mountPath: /app/secret
```

#### 🔹 2. HashiCorp Vault

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: vault-secret-demo
spec:
  volumes:
    - name: secret-volume
      emptyDir: { medium: "Memory" }   # in-memory only

  initContainers:
    - name: fetch-secret
      image: vault:1.13.0
      command:
        - sh
        - -c
        - |
          echo "Fetching secret from Vault..."
          vault login $VAULT_TOKEN
          vault kv get -field=password secret/myapp > /secret/password.txt
      env:
        - name: VAULT_ADDR
          value: "http://vault.default.svc.cluster.local:8200"
        - name: VAULT_TOKEN
          valueFrom:
            secretKeyRef:
              name: vault-token
              key: token
      volumeMounts:
        - name: secret-volume
          mountPath: /secret

  containers:
    - name: myapp
      image: nginx
      volumeMounts:
        - name: secret-volume
          mountPath: /app/secret
      env:
        - name: APP_PASSWORD_FILE
          value: /app/secret/password.txt
```

✅ **Usage**: Vault stores secrets at `secret/myapp`. Init container fetches them and writes into a shared volume.

---

#### 🔹 3. AWS Secrets Manager

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: aws-secret-demo
spec:
  volumes:
    - name: secret-volume
      emptyDir: { medium: "Memory" }

  initContainers:
    - name: fetch-secret
      image: amazon/aws-cli
      command:
        - sh
        - -c
        - |
          echo "Fetching secret from AWS Secrets Manager..."
          aws secretsmanager get-secret-value \
            --secret-id MyAppSecret \
            --query SecretString \
            --output text > /secret/app-secret.json
      env:
        - name: AWS_REGION
          value: us-east-1
      volumeMounts:
        - name: secret-volume
          mountPath: /secret

  containers:
    - name: myapp
      image: nginx
      volumeMounts:
        - name: secret-volume
          mountPath: /app/secret
```

* Main container can then parse `/secret/app-secret.json`.
* Use **Kubernetes Service Accounts + IAM Roles (IRSA / Workload Identity)** instead of hardcoding credentials.
* Ensure secrets are stored in **tmpfs (memory)** (`emptyDir: { medium: "Memory" }`) so they’re not written to disk.
* Set file permissions to **restrict access** (chmod 400).

---

## 🔹 4. Azure Key Vault

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: azure-secret-demo
spec:
  volumes:
    - name: secret-volume
      emptyDir: { medium: "Memory" }

  initContainers:
    - name: fetch-secret
      image: mcr.microsoft.com/azure-cli
      command:
        - sh
        - -c
        - |
          echo "Fetching secret from Azure Key Vault..."
          az login --identity   # uses Managed Identity
          az keyvault secret show \
            --vault-name MyKeyVault \
            --name MyAppSecret \
            --query value \
            -o tsv > /secret/app-secret.txt
      volumeMounts:
        - name: secret-volume
          mountPath: /secret

  containers:
    - name: myapp
      image: nginx
      volumeMounts:
        - name: secret-volume
          mountPath: /app/secret
```

✅ **Usage**: Uses **Managed Identity** to authenticate (no password stored in Kubernetes).

---

## 🔹 5. Google Cloud Secret Manager

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gcp-secret-demo
spec:
  volumes:
    - name: secret-volume
      emptyDir: { medium: "Memory" }

  initContainers:
    - name: fetch-secret
      image: google/cloud-sdk
      command:
        - sh
        - -c
        - |
          echo "Fetching secret from GCP Secret Manager..."
          gcloud secrets versions access latest \
            --secret="MyAppSecret" > /secret/app-secret.txt
      env:
        - name: GOOGLE_PROJECT
          value: my-gcp-project
      volumeMounts:
        - name: secret-volume
          mountPath: /secret

  containers:
    - name: myapp
      image: nginx
      volumeMounts:
        - name: secret-volume
          mountPath: /app/secret
```

✅ **Usage**: Pods authenticate with **Workload Identity** (maps Kubernetes SA to GCP IAM).

---

## 🔹 Common Best Practices

* Always use `emptyDir: { medium: "Memory" }` → secrets stored in memory only, not on disk.
* Use **Kubernetes Service Accounts + IAM roles** instead of static credentials.
* Apply **RBAC & NetworkPolicies** so only authorized pods can fetch secrets.
* Restrict access inside pod (`chmod 400` on secret files).
* Rotate secrets → Init Container fetches fresh ones every restart.

### References:
- https://devopscube.com/kubernetes-init-containers/