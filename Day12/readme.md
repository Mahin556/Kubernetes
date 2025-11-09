# Day 12/40 - Job and Cronjob in Kubernetes

## Check out the video below for Day12 👇

[![Day12/40 - Kubernetes job and Cronjob in Kubernetes](https://img.youtube.com/vi/kvITrySpy_k/sddefault.jpg)](https://youtu.be/kvITrySpy_k)


* Cron-job
```bash
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello
spec:
  schedule: "* * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox:1.28
            imagePullPolicy: IfNotPresent
            command:
            - /bin/sh
            - -c
            - date; echo Hello from the Kubernetes cluster
          restartPolicy: OnFailure
```

* Job
```bash
https://kubernetes.io/docs/concepts/workloads/controllers/job/
```