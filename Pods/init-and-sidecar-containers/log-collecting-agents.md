### References:- 
- https://spacelift.io/blog/kubernetes-sidecar-container

---
#### Logging Solutions
* https://www.elastic.co/logstash/
* https://www.fluentd.org/
* https://www.min.io/

---

* Fluentd
* get access key and secret access key
```bash
kubectl create namespace sidecar-logging-demo

kubectl config set-context $(kubectl config current-context) --namespace=sidecar-logging-demo

aws s3 mb s3://mahin-application-logs

aws s3 ls
```
```bash
touch fluentd-sidecar-config.yaml
```
* ConfigMap Explanation (fluentd-config)
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluentd-config
data:
  fluent.conf: |
    # First log source (tailing a file at /var/log/1.log)
    <source>
      @type tail
      format none
      path /var/log/1.log
      pos_file /var/log/1.log.pos
      tag count.format1
    </source>

    # Second log source (tailing a file at /var/log/2.log)
    <source>
      @type tail
      format none
      path /var/log/2.log
      pos_file /var/log/2.log.pos
      tag count.format2
    </source>

    # S3 output configuration (Store files every minute in the bucket's logs/ folder)
    <match **>
      @type s3
      aws_key_id <YOUR_AWS_ACCESS_KEY_ID> 
      aws_sec_key <YOUR_AWS_SECRET_ACCESS_KEY>
      s3_bucket <YOUR_S3_BUCKET_NAME>
      s3_region <YOUR_S3_BUCKET_REGION>
      path logs/
      buffer_path /var/log/
      store_as text
      time_slice_format %Y%m%d%H%M
      time_slice_wait 1m
      <instance_profile_credentials>
      </instance_profile_credentials>
    </match>
```
```bash
kubectl apply -f fluentd-sidecar-config.yaml
```
* **Detailed Explanation**
    * **`<source>` blocks:**
        Define *where Fluentd reads logs from*.
        * `/var/log/1.log` → tagged `count.format1`
        * `/var/log/2.log` → tagged `count.format2`
        * `pos_file` → keeps track of how much of the file has already been read, so Fluentd doesn’t re-ingest old logs after restarts.
        * `format none` → means each line is treated as plain text (no JSON or regex parsing).

    * **`<match>` block:**
        Defines *what Fluentd does with the collected data*.
        The pattern `**` means it matches *all tags* (`count.format1` and `count.format2` in this case).

        * `@type s3` → Send to Amazon S3.
        * `aws_key_id` and `aws_sec_key` → credentials for authentication.
        * `s3_bucket` and `s3_region` → your target bucket and AWS region.
        * `path logs/` → creates logs under `logs/` folder in the bucket.
        * `buffer_path` → temporary storage on the local filesystem for logs waiting to be uploaded.
        * `store_as text` → files will be uploaded as plain text.
        * `time_slice_format %Y%m%d%H%M` → files grouped per minute.
        * `time_slice_wait 1m` → wait 1 minute before uploading to S3 (to ensure all logs for that time slice are included).
        * `<instance_profile_credentials>` → optional block for IAM role-based authentication (useful if you’re on AWS EKS and don’t want to hardcode credentials).

```bash
touch multi-container-pod.yaml
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: counter
spec:
  containers:
  - name: count
    image: busybox
    command: ["/bin/sh", "-c"]
    args:
    - >
      i=0;
      while true;
      do
        # Write two log files along with the date and a counter
        # every second
        echo "$i: $(date)" >> /var/log/1.log;
        echo "$(date) INFO $i" >> /var/log/2.log;
        i=$((i+1));
        sleep 1;
      done
    # Mount the log directory /var/log using a volume
    volumeMounts:
    - name: varlog
      mountPath: /var/log
  - name: logging-agent
    image: lrakai/fluentd-s3:latest
    env:
    - name: FLUENTD_ARGS
      value: -c /fluentd/etc/fluent.conf
    # Mount the log directory /var/log using a volume
    # and the config file
    volumeMounts:
    - name: varlog
      mountPath: /var/log
    - name: config-volume
      mountPath: /fluentd/etc
  # Declare volumes for log directory and ConfigMap
  volumes:
  - name: varlog
    emptyDir: {}
  - name: config-volume
    configMap:
      name: fluentd-config
```
```bash
kubectl apply -f multi-container-pod.yaml
kubectl get pods
kubectl describe pod counter
kubectl logs -f counter -c logging-agent #Even if both containers show as Running, the sidecar may still have an error, for example due to incorrect Fluentd configuration or missing S3 credentials.
```
```bash
kubectl delete ns sidecar-logging-demo
```
---

* **Security Considerations**
    * ConfigMaps are **not** secure — anyone with access to the cluster can read them.
    * For production:
        * Use **Kubernetes Secrets** to store credentials.
        * Or use **IAM Roles for Service Accounts (IRSA)** in EKS (recommended).
        * Limit access to ConfigMaps with RBAC if you use them temporarily.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluentd-config
data:
  fluent.conf: |
    # First log source (tailing a file at /var/log/1.log)
    <source>
      @type tail
      format none
      path /var/log/1.log
      pos_file /var/log/1.log.pos
      tag count.format1
    </source>

    # Second log source (tailing a file at /var/log/2.log)
    <source>
      @type tail
      format none
      path /var/log/2.log
      pos_file /var/log/2.log.pos
      tag count.format2
    </source>

    # S3 output configuration (Store files every minute in the bucket's logs/ folder)
    <match **>
      @type s3
      aws_key_id "#{ENV['AWS_ACCESS_KEY_ID']}"
      aws_sec_key "#{ENV['AWS_SECRET_ACCESS_KEY']}"
      s3_bucket "#{ENV['AWS_S3_BUCKET']}"
      s3_region "#{ENV['AWS_REGION']}"
      path logs/
      buffer_path /var/log/
      store_as text
      time_slice_format %Y%m%d%H%M
      time_slice_wait 1m
      <instance_profile_credentials>
      </instance_profile_credentials>
    </match>
```
```bash
kubectl create secret generic aws-secret \
  --from-literal=access-key=YOUR_AWS_ACCESS_KEY \
  --from-literal=secret-key=YOUR_AWS_SECRET_KEY
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: counter
spec:
  containers:
  - name: count
    image: busybox
    command: ["/bin/sh", "-c"]
    args:
    - >
      i=0;
      while true;
      do
        # Write two log files along with the date and a counter
        # every second
        echo "$i: $(date)" >> /var/log/1.log;
        echo "$(date) INFO $i" >> /var/log/2.log;
        i=$((i+1));
        sleep 1;
      done
    # Mount the log directory /var/log using a volume
    volumeMounts:
    - name: varlog
      mountPath: /var/log
  - name: logging-agent
    image: lrakai/fluentd-s3:latest
    env:
    - name: FLUENTD_ARGS
      value: -c /fluentd/etc/fluent.conf
    # Mount the log directory /var/log using a volume
    # and the config file
    - name: AWS_ACCESS_KEY_ID
      valueFrom:
        secretKeyRef:
          name: aws-secret
          key: access-key
    - name: AWS_SECRET_ACCESS_KEY
      valueFrom:
        secretKeyRef:
          name: aws-secret
          key: secret-key
    - name: AWS_S3_BUCKET
      value: "mahin-application-logs"
    - name: AWS_REGION
      value: ap-south-1
    volumeMounts:
    - name: varlog
      mountPath: /var/log
    - name: config-volume
      mountPath: /fluentd/etc
    - name: fluentd-buffer
      mountPath: /fluentd/buffer
  # Declare volumes for log directory and ConfigMap
  volumes:
  - name: varlog
    emptyDir: {}
  - name: config-volume
    configMap:
      name: fluentd-config
  - name: fluentd-buffer
    emptyDir: {}
```
```bash
kubectl apply -f fluentd-config.yaml
```
Check logs of the Fluentd sidecar:
```bash
kubectl logs fluentd-sidecar-demo -c fluentd-sidecar
```




