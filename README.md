# IKB42603 Cloud Computing Security Essentials

## Lab 2 Report -- Secure Isolation & Multi-Tenancy

**Student:** Muhammad Akif Najmi bin Shamsul Bahar  
**Course:** IKB42603 Cloud Computing Security Essentials  
**Lab:** Lab 2 — Secure Isolation & Multi-Tenancy  
**Environment:** Docker, Kubernetes (kind), kubectl, Calico

---

## Objective

This lab demonstrates secure isolation in a multi-tenant cloud environment using Docker and Kubernetes.

The lab focuses on three major isolation dimensions:

- **Compute isolation** — separating tenants with Kubernetes namespaces and controlling shared resource consumption.
- **Network isolation** — applying a default-deny NetworkPolicy to prevent unauthorized cross-tenant traffic.
- **Storage isolation** — using Kubernetes RBAC to prevent one tenant from accessing another tenant's secrets.
- **Data remanence** — demonstrating why normal deletion does not necessarily guarantee secure erasure and showing a secure overwrite procedure.

The lab uses **Calico** as the Kubernetes network policy enforcement layer.

---

# Session A — Compute Isolation & Default-Open Risk

## Task 1 — Two Tenants on One Cluster

Two tenants were represented using separate Kubernetes namespaces:

- `tenant-a`
- `tenant-b`

Each tenant was deployed with its own NGINX web application and Kubernetes Service.

### Commands

```bash
kubectl create namespace tenant-a
kubectl create namespace tenant-b

kubectl -n tenant-a create deployment web --image=nginx
kubectl -n tenant-b create deployment web --image=nginx

kubectl -n tenant-a expose deployment web --port=80
kubectl -n tenant-b expose deployment web --port=80

kubectl get pods,svc -n tenant-a
kubectl get pods,svc -n tenant-b
```

### Result

Both tenants were successfully created on the same Kubernetes cluster while remaining logically separated by namespace.

The `tenant-b` web service was identified with the ClusterIP:

```text
10.96.251.249
```

The web application was running on the worker node with pod IP:

```text
192.168.19.66
```

### Security Significance

Namespaces provide logical separation, but they do **not automatically provide network isolation**. Additional security controls are required to prevent unauthorized communication between tenants.

**Status: COMPLETE**

---

# Task 2 — Observe the Default-Open Risk

Before applying any NetworkPolicy, a test pod in `tenant-a` was used to communicate with the web service belonging to `tenant-b`.

### Test

```bash
kubectl -n tenant-a run probe --rm -it \
  --image=curlimages/curl \
  --restart=Never \
  -- curl -s -m 5 http://10.96.251.249 \
  -o /dev/null -w '%{http_code}\n'
```

The test successfully reached the NGINX service and returned HTTP content.

The response was verified as an NGINX welcome page, demonstrating that traffic from one tenant could reach another tenant before network isolation was applied.

### Security Significance

This demonstrates the **default-open risk** of shared Kubernetes infrastructure.

Although `tenant-a` and `tenant-b` were placed in different namespaces, the namespaces alone did not prevent network communication.

In a real multi-tenant cloud environment, this could allow unintended access to services, APIs, or other workloads belonging to another customer.

**Status: COMPLETE**

---

# Task 3 — Contain the Noisy Neighbour with Resource Quotas

A Kubernetes `ResourceQuota` was applied to `tenant-a` to limit how many shared cluster resources the tenant could request.

### Resource Limits

| Resource | Limit |
|---|---:|
| CPU requests | 1 CPU |
| Memory requests | 512 MiB |
| Pods | 5 |

### Configuration

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: tenant-a-quota
  namespace: tenant-a
spec:
  hard:
    requests.cpu: "1"
    requests.memory: 512Mi
    pods: "5"
```

The quota was applied and checked with:

```bash
kubectl describe resourcequota tenant-a-quota -n tenant-a
```

### Security Significance

A resource quota prevents a single tenant from consuming an excessive amount of shared cluster capacity.

This addresses the **noisy-neighbour problem**, where one tenant could otherwise consume CPU, memory, or pod capacity and negatively affect other tenants.

**Status: COMPLETE**

---

# Session B — Network & Storage Isolation

## Task 4 — Default-Deny Network Isolation

A default-deny ingress NetworkPolicy was applied to `tenant-b`.

### NetworkPolicy

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: tenant-b
spec:
  podSelector: {}
  policyTypes:
    - Ingress
```

The policy was applied using:

```bash
kubectl apply -f -
```

The same connectivity test from Task 2 was then repeated.

### Test After NetworkPolicy

```bash
kubectl -n tenant-a run nettest \
  --rm -it \
  --restart=Never \
  --image=curlimages/curl \
  -- curl -v --max-time 5 http://10.96.251.249
```

The resulting output showed:

```text
* Trying 10.96.251.249:80...
* Connection timed out after 5002 milliseconds
curl: (28) Connection timed out after 5002 milliseconds
```

### Before vs After

| Test | Result |
|---|---|
| Before NetworkPolicy | HTTP connection successful |
| After NetworkPolicy | Connection timed out |

### Security Significance

The result demonstrates that the NetworkPolicy successfully changed the environment from **default-open communication** to **default-deny ingress** for `tenant-b`.

This follows the security principle:

> **Deny by default, permit by exception.**

The control prevents unauthorized cross-tenant network access and provides a stronger isolation boundary between workloads.

**Status: COMPLETE**

---

# Task 5 — Storage & Secret Isolation

Each tenant was given a Kubernetes Secret.

### Create Tenant Secrets

```bash
kubectl -n tenant-a create secret generic data \
  --from-literal=value=SECRET_A

kubectl -n tenant-b create secret generic data \
  --from-literal=value=SECRET_B
```

A service account was then created for `tenant-a`:

```bash
kubectl -n tenant-a create serviceaccount app-a
```

A Role allowing access to Secrets was created:

```bash
kubectl -n tenant-a create role reader \
  --verb=get \
  --resource=secrets
```

The Role was bound only to the `app-a` ServiceAccount:

```bash
kubectl -n tenant-a create rolebinding rb \
  --role=reader \
  --serviceaccount=tenant-a:app-a
```

The ServiceAccount identity was defined as:

```powershell
$SA="system:serviceaccount:tenant-a:app-a"
```

### RBAC Verification

```bash
kubectl auth can-i get secrets -n tenant-a --as=$SA
```

Result:

```text
yes
```

Cross-tenant access was then tested:

```bash
kubectl auth can-i get secrets -n tenant-b --as=$SA
```

Result:

```text
no
```

### Security Significance

The results prove that the ServiceAccount is authorized to access Secrets inside `tenant-a` but is not authorized to access Secrets in `tenant-b`.

This demonstrates **namespace-scoped RBAC** and the **Principle of Least Privilege**.

| Access Attempt | Result |
|---|---|
| `tenant-a` secrets | YES |
| `tenant-b` secrets | NO |

**Status: COMPLETE**

---

# Task 6 — Data Remanence & Secure Deletion

This task demonstrates that deleting a file normally does not necessarily guarantee that its underlying data is securely erased.

A Docker volume named `ccse-vol` was used.

## Normal Deletion Test

Sensitive data was created inside the mounted volume:

```bash
docker run --rm -v ccse-vol:/data alpine sh -c \
'echo SENSITIVE-PATIENT-RECORD > /data/phi.txt; sync; \
rm /data/phi.txt; \
grep -a SENSITIVE /data/* 2>/dev/null; \
echo scan-done'
```

The operation completed with:

```text
scan-done
```

This demonstrated the normal deletion/remanence test.

## Secure Wipe

A second file was created and overwritten with zero bytes before deletion:

```bash
docker run --rm -v ccse-vol:/data alpine sh -c \
'echo SENSITIVE > /data/phi2.txt; sync; \
dd if=/dev/zero of=/data/phi2.txt bs=1k count=1 conv=notrunc; \
rm /data/phi2.txt; \
echo wiped'
```

The command produced:

```text
1+0 records in
1+0 records out
1024 bytes (1.0KB) copied
wiped
```

### Security Significance

The experiment demonstrates the difference between **logical deletion** and **secure erasure**.

In cloud storage, administrators generally do not control the physical storage blocks directly. Therefore, cryptographic erasure is often the more practical cloud-security solution because destroying the encryption key makes the stored data computationally inaccessible.

**Status: COMPLETE**

---

# Verification

The lab manual recommends verifying the applied isolation controls with:

```bash
kubectl get networkpolicy -A
```

and:

```bash
kubectl describe resourcequota tenant-a-quota -n tenant-a
```

The lab environment also confirmed the relevant Kubernetes resources, namespaces, services, pods, NetworkPolicy, RBAC objects, and resource quota used throughout the exercises.

---

# Security Best-Practice Checklist

- [x] Tenants separated into distinct Kubernetes namespaces.
- [x] NGINX workloads deployed for both tenants.
- [x] Default-open cross-tenant communication demonstrated.
- [x] ResourceQuota used to limit shared resource consumption.
- [x] Default-deny NetworkPolicy applied.
- [x] Cross-tenant network connection blocked after policy enforcement.
- [x] Per-tenant Kubernetes Secrets created.
- [x] RBAC ServiceAccount scoped to `tenant-a`.
- [x] Tenant A secret access verified as allowed.
- [x] Tenant B secret access verified as denied.
- [x] Data-remanence test performed.
- [x] Secure overwrite and deletion demonstrated.

---

# Short-Answer Questions

## Q1. Why can containers in different namespaces reach each other by default, and why is that dangerous in multi-tenant cloud?

Kubernetes namespaces provide logical organization and resource separation, but they do not automatically block network traffic between namespaces. Without a NetworkPolicy, pods can normally communicate across namespaces.

This is dangerous in a multi-tenant cloud because one customer's workload may be able to reach services belonging to another customer. An attacker who compromises one workload could potentially use the network connection to discover or attack other workloads.

---

## Q2. Explain the default-deny principle and how your NetworkPolicy implements it.

The default-deny principle means that access is denied unless it has been explicitly permitted.

The NetworkPolicy applied to `tenant-b` selected all pods using:

```yaml
podSelector: {}
```

and specified:

```yaml
policyTypes:
  - Ingress
```

Therefore, incoming traffic to pods in `tenant-b` was denied by default. The same cross-tenant connection that previously succeeded subsequently timed out.

---

## Q3. How do virtual machines and containers differ in isolation strength? When would you add a VM boundary?

Containers normally share the host operating system kernel, making them lightweight and efficient but providing a weaker isolation boundary than virtual machines.

Virtual machines provide stronger isolation because each VM has its own operating-system kernel and is separated by a hypervisor.

A VM boundary would be appropriate when workloads have different trust levels, require stronger tenant isolation, process highly sensitive workloads, or when the risk of a container escape must be reduced.

---

## Q4. What is data remanence, and why is cryptographic erasure the preferred cloud solution?

Data remanence refers to residual information that can remain on storage media even after a file has been logically deleted.

In cloud environments, customers generally do not control the physical storage blocks. Therefore, physically overwriting storage is difficult to guarantee.

Cryptographic erasure is preferred because destroying the encryption key can make the remaining encrypted data computationally unusable without requiring physical access to the storage hardware.

---

## Q5. Which isolation dimension did each task exercise?

| Task | Isolation Dimension | Main Security Control |
|---|---|---|
| Task 1 | Compute | Kubernetes namespaces |
| Task 2 | Network | Demonstration of default-open communication |
| Task 3 | Compute / Resource | ResourceQuota |
| Task 4 | Network | NetworkPolicy |
| Task 5 | Storage | Kubernetes RBAC + Secrets |
| Task 6 | Storage / Data | Secure deletion / secure overwrite |

---

# Conclusion

Lab 2 successfully demonstrated the security challenges of multi-tenant cloud infrastructure and the controls required to address them.

The initial Kubernetes configuration showed that logical separation through namespaces alone does not automatically prevent cross-tenant communication. The default-open behaviour was successfully demonstrated by allowing a workload in `tenant-a` to reach the web service in `tenant-b`.

Resource quotas were then used to limit shared resource consumption and reduce the risk of a noisy neighbour affecting other tenants.

Network isolation was strengthened using a default-deny NetworkPolicy. The same connectivity test that previously succeeded subsequently timed out, providing clear evidence that the policy was being enforced.

Storage isolation was demonstrated using Kubernetes RBAC. The `tenant-a` ServiceAccount was allowed to access Secrets within its own namespace but was denied access to Secrets belonging to `tenant-b`.

Finally, the data-remanence experiment demonstrated the difference between normal deletion and secure wiping. The lab also highlighted why cryptographic erasure is the practical solution for many cloud-storage environments.

Overall, the lab demonstrated that effective multi-tenancy requires **layered isolation across compute, network, and storage**, rather than relying on namespace separation alone.

---

# Evidence Checklist

The following screenshots should be included in the GitHub repository under `screenshots/`:

| Evidence | File |
|---|---|
| Task 1 — Tenant A/B pods and services | `task1-tenants.png` |
| Task 2 — Before NetworkPolicy / successful connection | `task2-before-networkpolicy.png` |
| Task 3 — ResourceQuota verification | `task3-resourcequota.png` |
| Task 4 — After NetworkPolicy / timeout | `task4-networkpolicy-timeout.png` |
| Task 5 — RBAC `can-i` results | `task5-rbac-secret-isolation.png` |
| Task 6 — Data remanence scan | `task6-remanence.png` |
| Task 6 — Secure wipe | `task6-secure-wipe.png` |

Example GitHub evidence format:

```markdown
### Task 5 — Storage & Secret Isolation

![Task 5 RBAC Verification](screenshots/task5-rbac-secret-isolation.png)
```

---

# Technologies Used

- Docker
- Docker Desktop / Docker Engine
- Kubernetes
- kind
- kubectl
- Calico
- NGINX
- Kubernetes Namespaces
- Kubernetes Services
- Kubernetes ResourceQuota
- Kubernetes NetworkPolicy
- Kubernetes RBAC
- Kubernetes Secrets
- Alpine Linux
- curl

---

# References
1. Kubernetes Documentation — Network Policies.
2. Calico Documentation — Network Policy Enforcement.
3. Kubernetes Documentation — RBAC and Resource Management.
