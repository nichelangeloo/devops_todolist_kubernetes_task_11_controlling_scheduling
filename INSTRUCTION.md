## Kubernetes Controlling Scheduling Task

This project deploys:

- A MySQL StatefulSet in the `mysql` namespace.
- A Todo Application Deployment in the `todoapp` namespace.
- Supporting Kubernetes resources (Secrets, ConfigMaps, Services, PVCs, HPA, Ingress Controller).

The solution implements:

### MySQL StatefulSet

- Toleration for nodes tainted with:

    ```
    app=mysql:NoSchedule
    ```

- Node Affinity requiring scheduling on nodes labeled:

    ```
    app=mysql
    ```

- Pod Anti-Affinity preventing multiple MySQL replicas from running on the same node.

### TodoApp Deployment

- Preferred Node Affinity for nodes labeled:

    ```
    app=todoapp
    ```

- Pod Anti-Affinity preventing multiple TodoApp replicas from running on the same node.

---

# Deploy Resources

Execute:

```bash
./bootstrap.sh
```

Verify namespaces:

```bash
kubectl get ns
```

Expected:

```bash
mysql
todoapp
```

---

# Verify Node Labels

Check node labels:

```bash
kubectl get nodes --show-labels
```

Two nodes contain:

```bash
app=mysql
```

Three nodes contain:

```bash
app=todoapp
```

To inspect a specific node:

```bash
kubectl describe node <node-name>
```

---

# Verify MySQL Node Taint

Inspect node taints:

```bash
kubectl describe node <mysql-node>
```

Expected:

```bash
Taints:
app=mysql:NoSchedule
```

or

```bash
app=mysql:NoSchedule
```

---

# Verify StatefulSet Deployment

Check StatefulSet:

```bash
kubectl get statefulset -n mysql
```

Expected:

```bash
NAME    READY   AGE
mysql   2/2
```

Check pods:

```bash
kubectl get pods -n mysql -o wide
```

Example:

```bash
mysql-0   Running
mysql-1   Running
```

---

# Validate MySQL Toleration

Inspect pod specification:

```bash
kubectl get pod mysql-0 -n mysql -o yaml | grep -A5 tolerations
```

Expected:

```yaml
tolerations:
    - effect: NoSchedule
      key: app
      operator: Equal
      value: mysql
```

This confirms the pod can be scheduled on nodes tainted with:

```bash
app=mysql:NoSchedule
```

---

# Validate MySQL Node Affinity

Inspect affinity rules:

```bash
kubectl get pod mysql-0 -n mysql -o yaml | grep -A20 affinity
```

Expected:

```yaml
nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
```

with:

```yaml
key: app
values:
    - mysql
```

Verify actual node placement:

```bash
kubectl get pods -n mysql -o wide
```

Retrieve node labels:

```bash
kubectl get nodes --show-labels
```

Confirm MySQL pods are running only on nodes labeled:

```bash
app=mysql
```

---

# Validate MySQL Pod Anti-Affinity

List pod locations:

```bash
kubectl get pods -n mysql -o wide
```

Expected:

```bash
mysql-0   node-a
mysql-1   node-b
```

Pods should be scheduled on different nodes.

Verify anti-affinity configuration:

```bash
kubectl describe statefulset mysql -n mysql
```

Expected:

```yaml
podAntiAffinity: requiredDuringSchedulingIgnoredDuringExecution
```

using:

```yaml
topologyKey: kubernetes.io/hostname
```

---

# Verify TodoApp Deployment

Check deployment:

```bash
kubectl get deployment -n todoapp
```

Check pods:

```bash
kubectl get pods -n todoapp -o wide
```

Expected:

```bash
todoapp-xxxxx Running
todoapp-yyyyy Running
```

---

# Validate TodoApp Preferred Node Affinity

Inspect deployment:

```bash
kubectl describe deployment todoapp -n todoapp
```

Expected:

```yaml
preferredDuringSchedulingIgnoredDuringExecution
```

with:

```yaml
key: app
values:
    - todoapp
```

Check actual scheduling:

```bash
kubectl get pods -n todoapp -o wide
```

Verify pods are preferably placed on nodes labeled:

```bash
app=todoapp
```

---

# Validate TodoApp Pod Anti-Affinity

Inspect deployment:

```bash
kubectl describe deployment todoapp -n todoapp
```

Expected:

```yaml
podAntiAffinity: requiredDuringSchedulingIgnoredDuringExecution
```

Verify pod distribution:

```bash
kubectl get pods -n todoapp -o wide
```

Pods should be running on different nodes whenever cluster capacity allows.
