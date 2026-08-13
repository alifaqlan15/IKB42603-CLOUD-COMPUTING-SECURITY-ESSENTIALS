# Lab 2: Secure Isolation & Multi-Tenancy

## 1. Lab Overview

This lab demonstrates secure isolation and multi-tenancy in a cloud-native environment using Docker and Kubernetes.

The lab focuses on three isolation dimensions:

* Compute isolation
* Network isolation
* Storage isolation

The main activities include Kubernetes namespaces, resource quotas, NetworkPolicy, RBAC-based secret isolation, and secure deletion.

---

## 2. Learning Outcomes

At the end of this lab, the following concepts were demonstrated:

1. Compute isolation by separating tenants into Kubernetes namespaces.
2. The default-open behaviour of shared infrastructure and its security risks.
3. Network isolation using a default-deny NetworkPolicy.
4. Storage and secret isolation between tenants.
5. Data remanence and secure deletion.

## 3. Environment

### Tools Used

* Kali Linux
* Docker
* kind
* kubectl
* Kubernetes
* Calico

## 4. Lab Arrangement (2 Sessions over 2 Weeks)

| Session | Week | Focus |
| :--- | :--- | :--- |
| **Session A** | Week 3 | Compute isolation: containers, namespaces, resource quotas, and the default-open risk (Tasks 1–3) |
| **Session B** | Week 4 | Network & storage isolation: default-deny NetworkPolicy, per-tenant secrets, data remanence (Tasks 4–6), then the report |

> **Note:** Session A shows the problem (shared, open infrastructure). Session B applies the controls that make it safely separated. Keep outputs from both weeks for the report.

---

## 5.Technical Prerequisites

* A laptop with **at least 8 GB RAM** and admin rights.
* **Docker Desktop / Docker Engine** — free.
* **kind** and **kubectl** — free.
* Internet only for the first image download; the lab then runs offline.

---

## 6. Lab Arrangement (2 Sessions over 2 Weeks)

| Session | Week | Focus |
| :--- | :--- | :--- |
| **Session A** | Week 3 | Compute isolation: containers, namespaces, resource quotas, and the default-open risk (Tasks 1–3) |
| **Session B** | Week 4 | Network & storage isolation: default-deny NetworkPolicy, per-tenant secrets, data remanence (Tasks 4–6), then the report |

> **Note:** Session A shows the problem (shared, open infrastructure). Session B applies the controls that make it safely separated. Keep outputs from both weeks for the report.

---

## 7. Technical Prerequisites

* A laptop with **at least 8 GB RAM** and admin rights.
* **Docker Desktop / Docker Engine** — free.
* **kind** and **kubectl** — free.
* Internet only for the first image download; the lab then runs offline.

### Kubernetes Cluster

Cluster name :

```text
ccse-lab2

```

Pod Subnet :

```
192.168.0.0/16
```

The default kind CNI was disabled so that Calico could be used to enforce NetworkPolicy.

---

# Session A (Week 3) — Compute Isolation & the Default-Open Risk


## 1. Cluster Setup

### 1.1 Create Kubernetes Cluster

The cluster was created using kind with the default CNI disabled:

```bash
(echo kind: Cluster) > kind-config.yaml && (echo apiVersion: kind.x-k8s.io/v1alpha4) >> kind-config.yaml && (echo networking:) >> kind-config.yaml && (echo   disableDefaultCNI: true) >> kind-config.yaml && (echo   podSubnet: 192.168.0.0/16) >> kind-config.yaml

kind create cluster --name ccse-lab2 --config kind-config.yaml

```

The cluster was verified using:

```
kubectl get nodes
```

ccse-lab2-control-plane shown as *Ready*

Evidence :

<img width="777" height="82" alt="Image" src="https://github.com/user-attachments/assets/2bc116e8-34b7-4bab-a768-cb16f0790d59" />

### 1.2 Install Calico

Calico was installed as the cluster networking component:

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml

```

The Calico node was then verified :

```
kubectl -n kube-system rollout status daemonset/calico-node --timeout=180s

```
The Result :

```
daemon set "calico-node" successfully rolled out
```

Evidence :

<img width="933" height="956" alt="Image" src="https://github.com/user-attachments/assets/ff0965c4-788f-4f93-8e8f-fd3fbbc99d5d" />

Evidence :

<img width="930" height="83" alt="Image" src="https://github.com/user-attachments/assets/f09f8c5b-edaf-4a00-835e-5c36f120e210" />

---

## 2. Task 1 — Two Tenants on One Cluster

Two customers were modelled as two Kubernetes namespaces:

Deploying an nginx web server for each tenant:
```bash
kubectl create namespace tenant-a
kubectl create namespace tenant-b
```

Nginx web servers were deployed for both tenants:
```
kubectl -n tenant-a create deployment web --image=nginx
kubectl -n tenant-b create deployment web --image=nginx
```

Deploying a sample nginx web application for each tenant:
```
kubectl -n tenant-a expose deployment web --port=80
kubectl -n tenant-b expose deployment web --port=80
```

The pods and services were verified using:
```
kubectl get pods,svc -n tenant-a
kubectl get pods,svc -n tenant-b
```

This shows logical separation using Kubernetes namespaces on a shared cluster.

Evidence :

<img width="837" height="325" alt="Image" src="https://github.com/user-attachments/assets/58ea879c-9ca6-4727-98ef-7db1c3d8867e" />

---

## 3. Task 2 — Observe the Default Open Risk

Retrieve the tenant-b Service IP:
```
kubectl get svc web -n tenant-b -o jsonpath="{.spec.clusterIP}"
```
Test Cross-Tenant Connectivity:

```
kubectl -n tenant-a run probe --rm -it --image=curlimages/curl --restart=Never -- curl -s -m 5 http://<B_IP> -o /dev/null -w "HTTP %%{http_code}\n"
```
The Result :
```
HTTP 200
```
This verified that inter-tenant network traffic was permitted by default prior to enforcing a NetworkPolicy.

The outcome highlights the security risks of shared environments lacking explicit network controls.

Evidence :

<img width="922" height="71" alt="Image" src="https://github.com/user-attachments/assets/964ed076-6fd5-4698-b4ec-474803a42238" />

<img width="927" height="92" alt="Image" src="https://github.com/user-attachments/assets/924a8b0c-4749-4de8-984f-0e526bd4b4b1" />


## 4. Task 3 — Contain the Noisy Neighbour (Resource Quotas)

Create and apply the ResourceQuota for tenant-a:

```
(echo apiVersion: v1) > quota.yaml && (echo kind: ResourceQuota) >> quota.yaml && (echo metadata:) >> quota.yaml && (echo   name: tenant-a-quota) >> quota.yaml && (echo   namespace: tenant-a) >> quota.yaml && (echo spec:) >> quota.yaml && (echo   hard:) >> quota.yaml && (echo     requests.cpu: "1") >> quota.yaml && (echo     requests.memory: 512Mi) >> quota.yaml && (echo     pods: "5") >> quota.yaml

kubectl apply -f quota.yaml
```

Verify the applied quota:

```
kubectl describe resourcequota tenant-a-quota -n tenant-a
```

The quota enforces limits of 1 CPU request, 512 MiB memory request, and a maximum of 5 pods.
This ensures resource isolation and prevents noisy-neighbor issues between tenants.

Evidence :

<img width="933" height="471" alt="Image" src="https://github.com/user-attachments/assets/2db8efbc-46bd-412b-b458-d8d66909b847" />

---

# Session B (Week 4) — Network & Storage Isolation

## 5. Task 4 — Default-Deny Network Isolation

Apply the default-deny NetworkPolicy to tenant-a:

```
(echo apiVersion: networking.k8s.io/v1) > default-deny.yaml && (echo kind: NetworkPolicy) >> default-deny.yaml && (echo metadata:) >> default-deny.yaml && (echo   name: default-deny) >> default-deny.yaml && (echo   namespace: tenant-a) >> default-deny.yaml && (echo spec:) >> default-deny.yaml && (echo   podSelector: {}) >> default-deny.yaml && (echo   policyTypes:) >> default-deny.yaml && (echo   - Ingress) >> default-deny.yaml && (echo   - Egress) >> default-deny.yaml

kubectl apply -f default-deny.yaml
```

```
Test cross-tenant probe again to confirm traffic to tenant-b (10.96.220.23) is now blocked:
```

This confirms that cross-tenant communication was successfully blocked once the default-deny NetworkPolicy was enforced.

Evidence :

<img width="933" height="490" alt="Image" src="https://github.com/user-attachments/assets/705b3a4f-198d-42d9-90c2-87ae7bb12df3" />

## 6. Task 5 — Storage & Secret Isolation

Create Secrets & Service Account:

```
kubectl -n tenant-a create secret generic data --from-literal=value=SECRET_A
kubectl -n tenant-b create secret generic data --from-literal=value=SECRET_B
```

A service account scoped to tenant-a was created:

```
kubectl -n tenant-a create serviceaccount app-a
```

A Role allowing the service account to read secrets was created:
```
kubectl -n tenant-a create role reader --verb=get --resource=secrets
```

The Role was bound to the service account:

```
kubectl -n tenant-a create rolebinding rb --role=reader --serviceaccount=tenant-a:app-a
```

Verify RBAC Isolation:

```
kubectl auth can-i get secrets -n tenant-a --as=system:serviceaccount:tenant-a:app-a
```

The Result :

```
(Output must say yes)
```

The same service account was then tested against tenant-b:

```
kubectl auth can-i get secrets -n tenant-b --as=system:serviceaccount:tenant-a:app-a
```

This confirms that the tenant-a service account can read secrets in its own namespace while being blocked from accessing tenant-b secrets.

Evidence :

<img width="932" height="172" alt="Image" src="https://github.com/user-attachments/assets/f5e64b63-b7ee-441d-bc3e-fce89bf481d7" />

## 7. Task 6 — Data Remanence & Secure Deletion

Demonstrate Data Remanence (Standard Delete):

```
docker run --rm -v ccse-vol:/data alpine sh -c "echo SENSITIVE-PATIENT-RECORD > /data/phi.txt; sync; rm /data/phi.txt; grep -a SENSITIVE /data/* 2>/dev/null; echo scan-done"
```

Evidence :

<img width="941" height="280" alt="Image" src="https://github.com/user-attachments/assets/e9f8c6ba-c80d-428d-babf-052fac13f0c0" />

Demonstrate Secure Wipe (Overwrite Before Delete):

```
docker run --rm -v ccse-vol:/data alpine sh -c "echo SENSITIVE > /data/phi2.txt; sync; dd if=/dev/zero of=/data/phi2.txt bs=1k count=1 conv=notrunc; rm /data/phi2.txt; echo echo wiped"
```

Evidence :

<img width="928" height="170" alt="Image" src="https://github.com/user-attachments/assets/eadbb0a4-38ed-4344-ba43-bf50fb993266" />

---

## 8. Verify NetworkPolicy

The NetworkPolicy configuration was verified using:

```
kubectl get networkpolicy -A
```

The tenant-b namespace contained:

```
kubectl get networkpolicy -A
```

Evidence :

<img width="605" height="75" alt="Image" src="https://github.com/user-attachments/assets/8e6b82a8-0827-4e1d-8400-6f88cfc50233" />

---

## 9. Verify ResourceQuota

The ResourceQuota was verified using:

```
kubectl describe resourcequota tenant-a-quota -n tenant-a
```

Evidence :

<img width="926" height="211" alt="Image" src="https://github.com/user-attachments/assets/2ba990f8-0a8b-440d-af20-0bb3e7a5bd6d" />

---

## 10. Short-Answer Questions

**Q1. Why can containers in different namespaces reach each other by default, and why is that dangerous in multi-tenant cloud?**

By default, Kubernetes relies on a flat network model where namespaces only serve as logical organization tools, similar to folders, rather than strict network barriers. Every pod receives its own IP address and can freely communicate with any other pod across the cluster unless explicit network policies are applied to block it. This default behavior poses a severe security risk in a multi-tenant cloud environment because if an attacker compromises a pod belonging to Tenant A, they can easily scan and target pods owned by Tenant B. This exposes internal services and APIs that were designed under the assumption that internal cluster traffic is inherently safe. In Task 2, we observed this direct connectivity firsthand when Tenant A successfully called Tenant B's endpoint and received an HTTP 200 OK response without any restriction.

**Q2. Explain the default-deny principle and how your NetworkPolicy implements it.**

The default-deny principle represents a zero-trust networking strategy where all traffic is blocked by default, and access is only granted through explicit permission rules. In this lab, we implemented this strategy by applying a NetworkPolicy configured with an empty podSelector and setting the policyTypes to both Ingress and Egress. This configuration instructs Kubernetes to block all incoming and outgoing connections for every pod within that specific namespace. The policy proved effective when our test probe, which previously yielded an HTTP 200 status, immediately timed out and returned an HTTP 000 status once the policy was enforced.

Once applied, the cross-tenant probe that previously returned `HTTP 200` failed and returned `HTTP 000` (timeout), confirming default-deny network isolation.

**Q3. How do virtual machines and containers differ in isolation strength? When would you add a VM boundary?**

Containers are extremely lightweight because they share the host operating system kernel and rely solely on Linux namespaces and cgroups for process separation. However, if an attacker successfully exploits a vulnerability within the kernel from inside a container, they can potentially compromise the entire host system. Virtual Machines, on the other hand, run a full guest operating system on top of a hypervisor, providing hardware-level isolation that is significantly stronger. You should introduce a VM boundary or use sandboxed container runtimes like Kata Containers whenever you execute untrusted code, host high-risk multi-tenant environments, or handle strict compliance workloads that forbid kernel sharing.

**Q4. What is data remanence, and why is cryptographic erasure the preferred cloud solution?**

Data remanence refers to the residual data that remains on a storage device even after a file has been deleted through standard operating system procedures. Typical file deletion only removes the index pointer while leaving the underlying raw data on the physical drive until it eventually gets overwritten. In a cloud environment, managing this is difficult because users do not have physical access or control over the underlying drives, making traditional overwriting tools like the dd command impractical across shared storage platforms. Cryptographic erasure solves this by encrypting the data at rest and simply destroying the decryption key when deletion is required, instantly rendering any remaining raw data on the drive completely unreadable.

**Q5. Which of the three isolation dimensions (compute, network, storage) did each task exercise?**

| Task | Isolation Dimension | Demonstration |
| :--- | :--- | :--- |
| **Task 1** | Compute | Tenants were separated using Kubernetes namespaces. |
| **Task 2** | Network | Default-open cross-tenant communication was demonstrated (`HTTP 200`). |
| **Task 3** | Compute | `ResourceQuota` limited shared CPU, memory, and pod resource consumption. |
| **Task 4** | Network | `default-deny` NetworkPolicy blocked cross-tenant traffic (`HTTP 000`). |
| **Task 5** | Storage | RBAC restricted secret access between tenants (`auth can-i`). |
| **Task 6** | Storage | Data remanence and secure deletion were demonstrated using container volumes. |

---

## 11. Security Best-Practices Checklist

- [x] Tenants are separated into distinct namespaces.
- [x] A default-deny `NetworkPolicy` blocks cross-tenant traffic and was verified before and after.
- [x] Resource quotas prevent a noisy-neighbour from exhausting shared capacity.
- [x] Per-tenant secrets are unreadable by other tenants through RBAC.
- [x] Secure deletion and cryptographic erasure are understood for data remanence.

---

## 12. Conclusion

This lab provided hands-on experience in building a secure multi-tenant environment on Kubernetes by implementing defense-in-depth security controls. The practical exercises clearly proved that native Kubernetes installations are open and insecure out of the box, relying entirely on administrators to enforce strict isolation boundaries across compute, network, and storage layers.

By combining logical namespaces and resource quotas, we successfully prevented resource starvation and neutralized noisy-neighbor threats. Network policies enforced a strict zero-trust posture by replacing default cross-tenant connectivity with default-deny rules, ensuring pods can only communicate when explicitly authorized. Finally, leveraging role-based access control secured sensitive secrets, while cryptographic erasure addressed the critical reality of cloud data remanence where physical disk access is unavailable. Ultimately, these multi-layered security practices demonstrate that robust tenant isolation in modern cloud infrastructure requires active, end-to-end enforcement rather than relying on default system configurations.

---

## 13. Cleanup & Teardown

After all verification tests, screenshots, and documentation were completed, the lab environment was completely torn down using:

```bash
kind delete cluster --name ccse-lab2
docker volume rm ccse-vol
```


## 14. References

1. Course lecture — Week 3 (Secure Isolation of Physical & Logical Infrastructure).
2. Kubernetes Network Policies — kubernetes.io/docs/concepts/services-networking/networkpolicies
3. Calico documentation — docs.tigera.io
4. CSA Security Guidance v5 — Infrastructure & Networking domain





























