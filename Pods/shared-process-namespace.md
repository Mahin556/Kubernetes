```yaml
apiVersion: v1
kind: Pod
metadata:
  name: shared-pid-pod
spec:
  shareProcessNamespace: true
  containers:
  - name: main-app
    image: my-app-image
  - name: debugger-sidecar
    image: busybox:1.28
    command: ["sleep", "3600"]
    securityContext:
      capabilities:
        add: ["SYS_PTRACE"] # Required for advanced debugging like strace
    stdin: true
    tty: true
```
```bash
controlplane:~$ kubectl exec -it shared-pid-pod -c debugger-sidecar -- sh
/ # ps
PID   USER     TIME  COMMAND
    1 65535     0:00 /pause
    7 root      0:00 nginx: master process nginx -g daemon off;
   35 101       0:00 nginx: worker process
   36 root      0:00 sleep 3600
   42 root      0:00 sh
   48 root      0:00 ps
/ # 
```
* `ps -ef | grep [m]ain-app | awk '{print $1}'`
* `lsns -p <PID>`


