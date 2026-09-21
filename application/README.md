# Kubernetes Lab Session

This lab introduces Kubernetes using **K3s**. We will create Pods, Deployments, Services, and Namespaces using both **Imperative Commands** and **Declarative YAML files**.

## Lab Architecture

```yaml
User
  |
  |  NodePort
  v
Service
  |
  v
Deployment
  |
  +--------+--------+
  |        |        |
 Pod 1    Pod 2    Pod 3
  |
  v
Highway / WaveCafe Application
```

---

# 1. Install K3s Kubernetes

Update the Ubuntu server:

```bash
sudo apt update
```

Install K3s:

```bash
curl -sfL https://get.k3s.io | sh -
```

Verify the Kubernetes node:

```bash
kubectl get nodes
```

Expected result:

```c#
NAME        STATUS   ROLES                  AGE   VERSION
server      Ready    control-plane,master   1m    v1.x.x
```

Check Pods:

```bash
kubectl get pods
```

---

# 2. Kubernetes Namespaces

List available namespaces:

```bash
kubectl get namespace
```

Short form:

```bash
kubectl get ns
```

Create a namespace called `dev`:

```bash
kubectl create namespace dev
```

Verify:

```bash
kubectl get ns
```

---

# 3. Imperative Approach – Create a Pod

In the imperative approach, we create Kubernetes resources directly using commands.

Create a Pod using the Highway Docker image:

```bash
kubectl run highway-pod \
  --image=nextkodeschool/highway:latest \
  --port=80
```

Check the Pod:

```bash
kubectl get pods
```

View more information:

```bash
kubectl get pods -o wide
```

Delete the Pod:

```bash
kubectl delete pod highway-pod
```

Verify:

```bash
kubectl get pods
```

> A standalone Pod does not automatically come back after we delete it.

---

# 4. Imperative Approach – Create a Deployment

Create a Deployment inside the `dev` namespace:

```bash
kubectl create deployment highway-deployment \
  --image=nextkodeschool/highway:latest \
  -n dev
```

Check Pods in the default namespace:

```bash
kubectl get pods
```

The Highway Pod will not appear here because it was created in the `dev` namespace.

Check Pods in `dev`:

```bash
kubectl get pods -n dev
```

Check the Deployment:

```bash
kubectl get deployment -n dev
```

Short form:

```bash
kubectl get deploy -n dev
```

---

# 5. Scale the Application

Currently, the Deployment creates one Pod.

Scale it to **3 replicas**:

```bash
kubectl scale deployment highway-deployment \
  -n dev \
  --replicas=3
```

Verify the Deployment:

```bash
kubectl get deployment -n dev
```

Check the Pods:

```bash
kubectl get pods -n dev
```

You should now see three Pods:

```c#
highway-deployment-xxxxx-aaaaa
highway-deployment-xxxxx-bbbbb
highway-deployment-xxxxx-ccccc
```

---

# 6. Test Kubernetes Self-Healing

Get the Pod names:

```bash
kubectl get pods -n dev
```

Choose one Pod and delete it:

```json
kubectl delete pod <pod-name> -n dev
```

Example:

```json
kubectl delete pod highway-deployment-66f655544f-dllkj -n dev
```

Immediately check the Pods:

```bash
kubectl get pods -n dev
```

The Deployment detects that only two Pods are running and automatically creates another Pod.

```c#
Desired Replicas = 3

Pod 1   Running
Pod 2   Running
Pod 3   Deleted
          |
          v
Deployment creates replacement Pod
          |
          v
Pod 4   Running

Total Running Pods = 3
```

This demonstrates Kubernetes **self-healing**.

---

# 7. Expose the Deployment Using NodePort

The Pods are running, but we need a Service to expose the application.

Create a NodePort Service:

```bash
kubectl expose deployment highway-deployment \
  --type=NodePort \
  --port=80 \
  --target-port=80 \
  --name=highway-service \
  -n dev
```

Check the Service:

```bash
kubectl get service -n dev
```

Or:

```bash
kubectl get svc -n dev
```

Example:

```c#
NAME              TYPE       CLUSTER-IP      PORT(S)
highway-service   NodePort   10.43.100.10    80:31234/TCP
```

In this example:

```c#
80     = Service Port
31234  = NodePort
```

Kubernetes automatically assigns a NodePort when `--node-port` is not explicitly specified.

---

# 8. Access the Application

Find the assigned NodePort:

```bash
kubectl get svc -n dev
```

Find the server's public IP address.

Access the application using:

```c#
http://<PUBLIC-IP>:<NODEPORT>
```

Example:

```c#
http://54.x.x.x:31234
```

If this server is running on AWS EC2, make sure the assigned NodePort is allowed in the **EC2 Security Group inbound rules**.

---

# 9. Imperative Approach Summary

So far, we created resources directly using commands:

```bash
# Namespace
kubectl create namespace dev

# Pod
kubectl run highway-pod \
  --image=nextkodeschool/highway:latest \
  --port=80

# Deployment
kubectl create deployment highway-deployment \
  --image=nextkodeschool/highway:latest \
  -n dev

# Scale
kubectl scale deployment highway-deployment \
  --replicas=3 \
  -n dev

# Service
kubectl expose deployment highway-deployment \
  --type=NodePort \
  --port=80 \
  --target-port=80 \
  --name=highway-service \
  -n dev
```

This is called the **Imperative Approach**.

---

# 10. Declarative Approach

In the declarative approach, Kubernetes resources are defined using YAML files.

We will use:

```c#
application/
├── deployment.yaml
└── service.yaml
```

Clone the Kubernetes repository:

```bash
git clone https://github.com/nextkodeschool/Kubernetes.git
```

Navigate to the application directory:

```bash
cd Kubernetes/application/
```

Check the files:

```bash
ls
```

---

# 11. deployment.yaml

Example Deployment:

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: wavecafe-deployment
  namespace: dev

spec:
  replicas: 2

  selector:
    matchLabels:
      app: wavecafe

  template:
    metadata:
      labels:
        app: wavecafe

    spec:
      containers:
        - name: wavecafe
          image: nextkodeschool/wavecafe:latest
          imagePullPolicy: Always

          ports:
            - containerPort: 80
```

Create the Deployment:

```bash
kubectl create -f deployment.yaml
```

Check the Pods:

```bash
kubectl get pods -n dev
```

Check the Deployment:

```bash
kubectl get deployment -n dev
```

---

# 12. service.yaml

Create `service.yaml`:

```yaml
apiVersion: v1
kind: Service

metadata:
  name: wavecafe-service
  namespace: dev

spec:
  type: NodePort

  selector:
    app: wavecafe

  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 30080
```

Create the Service:

```bash
kubectl create -f service.yaml
```

Verify:

```bash
kubectl get svc -n dev
```

The application can now be accessed using:

```c#
http://<NODE-PUBLIC-IP>:30080
```

---

# 13. Understanding Kubernetes Ports

In our Service:

```yaml
ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
```

Traffic flows like this:

```c#
Browser
   |
   | :30080
   v
Kubernetes Node
   |
   v
NodePort Service
   |
   | port: 80
   v
Service
   |
   | targetPort: 80
   v
Pod / Container
   |
   v
Apache Website
```

### `port`

```yaml
port: 80
```

Port exposed by the Kubernetes **Service inside the cluster**.

### `targetPort`

```yaml
targetPort: 80
```

Port where the application is actually listening **inside the Pod/container**.

### `nodePort`

```yaml
nodePort: 30080
```

Port exposed on the Kubernetes **Node** so users outside the cluster can reach the application.

---

# 14. Update a Deployment

Edit the Deployment:

```bash
vim deployment.yaml
```

For example, change:

```yaml
replicas: 2
```

to:

```yaml
replicas: 3
```

Apply the changes:

```bash
kubectl apply -f deployment.yaml
```

Verify:

```bash
kubectl get pods -n dev
```

You should now see three Pods.

---

# 15. `create` vs `apply`

Create a resource for the first time:

```bash
kubectl create -f deployment.yaml
```

For declarative management, the commonly used command is:

```bash
kubectl apply -f deployment.yaml
```

`apply` can create the resource if it doesn't exist and update it when the YAML configuration changes.

Example:

```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```

---

# 16. Delete Resources Using YAML

Delete the Deployment:

```json
kubectl delete -f deployment.yaml
```

Delete the Service:

```json
kubectl delete -f service.yaml
```

Or delete both:

```json
kubectl delete -f deployment.yaml -f service.yaml
```

Verify:

```bash
kubectl get all -n dev
```

---

# 17. Useful Debugging Commands

Check Pods:

```bash
kubectl get pods -n dev
```

Get additional information:

```bash
kubectl get pods -n dev -o wide
```

Describe a Pod:

```bash
kubectl describe pod <pod-name> -n dev
```

Check container logs:

```bash
kubectl logs <pod-name> -n dev
```

Check Deployments:

```bash
kubectl get deployments -n dev
```

Describe a Deployment:

```bash
kubectl describe deployment wavecafe-deployment -n dev
```

Check Services:

```bash
kubectl get svc -n dev
```

Describe the Service:

```bash
kubectl describe service wavecafe-service -n dev
```

Check all major resources:

```bash
kubectl get all -n dev
```

---

# 18. Important kubectl Short Forms

```c#
pods         → po
deployments  → deploy
services     → svc
namespaces   → ns
```

Examples:

```bash
kubectl get po -n dev
kubectl get deploy -n dev
kubectl get svc -n dev
kubectl get ns
```

---

# 19. Lab Summary

In this lab, we learned:

```c#
K3s Installation
      |
      v
Create Namespace
      |
      v
Create Pod
      |
      v
Create Deployment
      |
      v
Scale Deployment
      |
      v
Test Self-Healing
      |
      v
Create NodePort Service
      |
      v
Access Application
      |
      v
Create Resources Using YAML
      |
      v
Update & Delete Resources
```

We practiced both Kubernetes management approaches:

**Imperative**

```bash
kubectl run
kubectl create
kubectl scale
kubectl expose
kubectl delete
```

**Declarative**

```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```

The **declarative YAML approach** is commonly preferred for repeatable application deployments because the desired configuration can be stored and version-controlled in Git.
