### References:
- https://www.tutorialspoint.com/docker/docker_working_of_kubernetes.htm

---

### **1. Docker and Kubernetes Overview**

* **Docker**: Packages an application and all its dependencies into a **container**, ensuring consistency across **development, staging, testing, and production** environments.

* **Kubernetes**: Orchestrates containers at scale, providing:

  * Automated **scheduling** and placement of containers
  * **Scaling** up or down based on load
  * **Self-healing** for failed containers
  * **Load balancing** across pods
  * **Declarative deployments** using YAML manifests

* **Cluster Architecture**:

  * **Master Node**: Manages the cluster state and schedules workloads.
  * **Worker Nodes**: Run the containerized applications.
  * **Pods**: Smallest deployable units in Kubernetes, containing one or more containers.

---

### **2. Preparing a Java Spring Boot Application**

1. **Generate a Spring Boot Project**

   * Use [Spring Initializr](https://start.spring.io/) to create a Maven/Gradle project.
   * Add dependencies like **Spring Web** for REST API support.

2. **Create a REST Controller**

```java
package com.example.demo;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class HelloController {
   @GetMapping("/")
   public String hello() {
      return "Hello from Spring Boot!";
   }
}
```

3. **Build the Application**

   ```bash
   mvn clean package
   ```

   * Output: JAR file in `target/` directory.
   * Best practices:

     * **Dependency management** via Maven/Gradle
     * **Externalize configuration** using properties or environment variables
     * Write **unit and integration tests**

---

### **3. Building a Docker Image**

1. **Create a Dockerfile**

```dockerfile
FROM openjdk:17-jdk-alpine
WORKDIR /app
COPY target/*.jar app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]
```

* **FROM openjdk:17-jdk-alpine**: Base image
* **WORKDIR /app**: Sets working directory in container
* **COPY target/*.jar app.jar**: Copies the application JAR
* **ENTRYPOINT**: Command to run the JAR

2. **Build the Docker Image**

```bash
docker build -t my-spring-boot-app .
```

3. **Verify Docker Image**

```bash
docker images
```

* Best practices:

  * Choose **lightweight base images** (e.g., Alpine)
  * Use **multi-stage builds** to reduce image size
  * Follow **Dockerfile best practices** to optimize performance

---

### **4. Kubernetes Setup (Minikube)**

1. **Install Minikube**

```bash
curl -L https://github.com/minikube/minikube/releases/latest/download/minikube-linux-amd64 > minikube
chmod +x minikube
sudo mv minikube /usr/local/bin/
minikube start
```

2. **Install Kubectl**

```bash
curl -LO https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
```

* Configure:

```bash
minikube config set context minikube
kubectl cluster-info
minikube status
kubectl get nodes
```

* Best practices:

  * Consider **other Kubernetes distributions**: K3s, MicroK8s
  * For production, use **cloud-managed Kubernetes**: GKE, EKS, AKS
  * Configure **node resources** according to app requirements

---

### **5. Kubernetes Deployment**

1. **Deployment YAML**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
   name: my-spring-boot-app
spec:
   replicas: 3
   selector:
      matchLabels:
         app: my-spring-boot-app
   template:
      metadata:
         labels:
            app: my-spring-boot-app
      spec:
         containers:
         - name: my-spring-boot-app
           image: my-spring-boot-app:latest
           ports:
           - containerPort: 8080
```

* **replicas**: Number of pod instances
* **matchLabels**: Ensures the Deployment manages the correct pods
* **containerPort**: Port the app listens on inside container

2. **Service YAML**

```yaml
apiVersion: v1
kind: Service
metadata:
   name: my-spring-boot-app-service
spec:
   type: NodePort
   selector:
      app: my-spring-boot-app
   ports:
      - protocol: TCP
        port: 80
        targetPort: 8080
```

* **type: NodePort** exposes service outside the cluster
* **port**: Exposed by service
* **targetPort**: Forwards traffic to pods

3. **Apply Configurations**

```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```

4. **Verify Deployment**

```bash
kubectl get pods
kubectl get services
```

5. **Access Application**

```bash
minikube service my-spring-boot-app-service
```

* Best practices:

  * Adjust **replica count** for scaling
  * Add **security contexts** and **network policies**
  * Implement **monitoring** using Prometheus/Grafana

---

### **6. How Kubernetes Orchestrates Docker Containers**

* Pods run Docker containers.
* Deployments ensure the **desired state** (replicas, container image versions).
* Services provide **stable networking**.
* Kubernetes automatically handles:

  * **Rollouts** and **updates**
  * **Scaling** of pods
  * **Self-healing** of failed containers
  * **Load balancing**

---

### **7. Summary**

* Docker ensures **consistency** of the application environment.
* Kubernetes handles **orchestration** of multiple containers.
* YAML manifests define **desired state** for deployments and services.
* Minikube provides a **local Kubernetes environment** for testing and development.
* Best practices include **scaling, security, and monitoring** for production readiness.

---

### **8. FAQs**

1. **Difference between Docker and Kubernetes?**

   * Docker: Builds, runs, and manages containers.
   * Kubernetes: Orchestrates multiple containers across nodes for scaling, load balancing, and self-healing.

2. **How does Kubernetes orchestrate containers?**

   * Pods group containers.
   * Deployments manage replica sets.
   * Services provide stable networking.
   * Kubernetes automates lifecycle management, scaling, and load balancing.

---

If you want, I can also **draw a full workflow diagram showing Docker → Kubernetes → Spring Boot deployment** with Pods, Deployment, and Service interactions. This helps visualize the orchestration process.

Do you want me to make that diagram?
