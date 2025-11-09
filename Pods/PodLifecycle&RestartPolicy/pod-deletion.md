## 📘 **References**
* [Kubernetes: Pod Lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)
* [Kubernetes: Container Images](https://kubernetes.io/docs/concepts/containers/images/)

---

![Alt text](/images/22a.png)

```bash
controlplane:~$ kill -l
 1) SIGHUP       2) SIGINT       3) SIGQUIT      4) SIGILL       5) SIGTRAP
 6) SIGABRT      7) SIGBUS       8) SIGFPE       9) SIGKILL     10) SIGUSR1
11) SIGSEGV     12) SIGUSR2     13) SIGPIPE     14) SIGALRM     15) SIGTERM
16) SIGSTKFLT   17) SIGCHLD     18) SIGCONT     19) SIGSTOP     20) SIGTSTP
21) SIGTTIN     22) SIGTTOU     23) SIGURG      24) SIGXCPU     25) SIGXFSZ
26) SIGVTALRM   27) SIGPROF     28) SIGWINCH    29) SIGIO       30) SIGPWR
31) SIGSYS      34) SIGRTMIN    35) SIGRTMIN+1  36) SIGRTMIN+2  37) SIGRTMIN+3
38) SIGRTMIN+4  39) SIGRTMIN+5  40) SIGRTMIN+6  41) SIGRTMIN+7  42) SIGRTMIN+8
43) SIGRTMIN+9  44) SIGRTMIN+10 45) SIGRTMIN+11 46) SIGRTMIN+12 47) SIGRTMIN+13
48) SIGRTMIN+14 49) SIGRTMIN+15 50) SIGRTMAX-14 51) SIGRTMAX-13 52) SIGRTMAX-12
53) SIGRTMAX-11 54) SIGRTMAX-10 55) SIGRTMAX-9  56) SIGRTMAX-8  57) SIGRTMAX-7
58) SIGRTMAX-6  59) SIGRTMAX-5  60) SIGRTMAX-4  61) SIGRTMAX-3  62) SIGRTMAX-2
63) SIGRTMAX-1  64) SIGRTMAX
```

<br>

* **Graceful Delete (Default)**
    ```bash
    kubectl delete pod <pod-name>
    ```
    * Sends **SIGTERM**, waits for `terminationGracePeriodSeconds` (default 30s).
    * If still running after grace period → sends **SIGKILL**.
    * Pod object deleted from API server **after container stops**.

<br>

* **Immediate API Deletion (Still Graceful on Node)**
    ```bash
    kubectl delete pod <pod-name> --force=true
    ```
    * **Removes Pod object immediately** from API server.
    * **Kubelet still honors** the Pod’s termination grace period.
    * Useful if API cleanup is urgent but you still want graceful shutdown.

<br>

* **Forceful Immediate Deletion (SIGKILL)**
    ```bash
    kubectl delete pod <pod-name> --force=true --grace-period=0
    ```
    * Pod deleted from API server **instantly**.
    * Kubelet **stops tracking** it and sends **SIGKILL** to containers immediately.
    * No cleanup or waiting — used for **hung/stuck Pods**.

<br>

* **Signal Flow Summary**

    | Signal              | Sent By        | Purpose                                           |
    | ------------------- | -------------- | ------------------------------------------------- |
    | **SIGTERM**         | Kubelet        | Request app to gracefully shut down               |
    | **SIGKILL**         | Kubelet        | Force kill if app doesn’t exit in time            |
    | **SIGTERM Handler** | App (optional) | Clean up resources, close connections, flush data |

<br>

* **Bonus: Check Termination Events**
    ```bash
    kubectl describe pod <pod-name> | grep -A5 "State:"
    ```
    * See if termination reason was:
        * `Completed`
        * `Error`
        * `OOMKilled`
        * `Signal: 9` (force killed)





