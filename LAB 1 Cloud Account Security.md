# IKB42603 Cloud Computing Security Essentials

## LAB 1 WEEKS 1-2: Cloud Account Security, Identity & Access Management

**Identity governance and least privilege: LocalStack IAM & Kubernetes RBAC**

---

### Course & Assessment Mapping

| Item | Mapping |
|---|---|
| **Course Learning Outcome** | CLO2 – Construct secure cloud operations that safeguard data integrity |
| **Lecture topics** | Weeks 1-2 (Fundamentals, Security Architecture); Weeks 5 & 7 (Access Control, Identity) |
| **Value / skill clusters** | VBE3 (Integrity); SC8 (Integrated Problem-Solving) |
| **Assessment** | Lab report (screenshots + CLI output + short answers), contributes to the Lab Assignment |

---

### Lab Arrangement (2 Sessions over 2 Weeks)

| Session | Week | Focus |
|---|---|---|
| **Session A** | Week 1 | Environment setup + cloud identity with LocalStack IAM (Tasks 1–4) |
| **Session B** | Week 2 | Enforced access control with Kubernetes RBAC + audit (Tasks 5–7), then the report |

> **Note:** Session A was completed fully before starting Session B. Commands and screenshots were kept throughout, as required for this report.

---

### Technical Prerequisites

- A laptop with at least 8 GB RAM and administrator rights to install software
- Docker Desktop / Docker Engine
- AWS CLI v2 (pointed at LocalStack, not real AWS)
- `kind` (Kubernetes-in-Docker) and `kubectl`
- Internet access only for the first download of container images

> **Security tip:** Nothing in this lab connects to a real cloud provider — LocalStack emulates AWS APIs locally, and `kind` runs Kubernetes inside Docker on this machine.

---

# Session A (Week 1): Cloud Identity with LocalStack

## One-Time Environment Setup

```bash
# 1. Confirm Docker is installed and running
docker --version

# 2. Start LocalStack (AWS-compatible cloud) in a container
docker run -d --name localstack -p 4566:4566 localstack/localstack:2.3.2

# 3. Confirm it is healthy (should list running services)
curl http://localhost:4566/_localstack/health

# Configure dummy credentials (LocalStack accepts any value)
aws configure set aws_access_key_id test
aws configure set aws_secret_access_key test
aws configure set region us-east-1

# Test: this talks to LocalStack, NOT real AWS
aws --endpoint-url=http://localhost:4566 sts get-caller-identity

```
**Evidence :**


<img width="308" height="47" alt="Image" src="https://github.com/user-attachments/assets/99f0820f-86c2-4a63-999d-504a0180045f" />

<img width="663" height="55" alt="Image" src="https://github.com/user-attachments/assets/396a9543-3df4-4346-afc9-c67985279fb2" />

---

## Task 1: Map the Cloud Identity Landscape

| Concept | AWS term | Purpose |
|---|---|---|
| **All-powerful owner** | Root user | The identity created automatically when the account is opened. It has unrestricted access to every resource and billing setting, which is exactly why it should never be used for everyday work — it should be locked away (MFA-protected) and reserved for account-level emergencies only. |
| **Human/app identity** | IAM User | A named identity for a specific person or application that needs to authenticate individually, typically with its own password and/or access keys. |
| **Permission bundle** | IAM Policy | A JSON document that states exactly which actions are allowed or denied on which resources. Policies are how least privilege is actually expressed in AWS. |
| **Collection of users** | IAM Group | A container of IAM users that lets an administrator manage permissions for many people at once by attaching a policy to the group instead of to each user. |
| **Temporary identity** | IAM Role | An identity with no long-term credentials of its own — it is *assumed* (by a user, application, or AWS service) for a limited session, after which the temporary credentials expire. |

**Evidence :**

<img width="401" height="87" alt="Image" src="https://github.com/user-attachments/assets/59403390-9f62-43a6-a892-5e3f9b8fbcec" />

<img width="608" height="107" alt="Image" src="https://github.com/user-attachments/assets/890d3cde-0b03-4be3-9d25-f6dee843d5cd" />

---

## Task 2: Create a Least-Privilege Admin (Stop Using Root)

```
EP='--endpoint-url=http://localhost:4566'

# 2.1 Create a group and attach an admin policy to the GROUP
aws $EP iam create-group --group-name Admins
aws $EP iam attach-group-policy --group-name Admins \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess
  
# 2.2 Create a personal admin user
aws $EP iam create-user --user-name CloudAdmin_AlifAqlan

# 2.3 Put the user in the group (permissions flow from the group)
aws $EP iam add-user-to-group --group-name Admins \
  --user-name CloudAdmin_AlifAqlan
  
# 2.4 Verify the membership
aws $EP iam get-group --group-name Admins
```

**Evidence :**

<img width="397" height="35" alt="Image" src="https://github.com/user-attachments/assets/768d942d-ad3f-4e92-a60f-f7f6ea634221" />

<img width="527" height="231" alt="Image" src="https://github.com/user-attachments/assets/80d2ecd9-881f-4dcb-8833-ea607c3c4f27" />

<img width="487" height="55" alt="Image" src="https://github.com/user-attachments/assets/135c8a19-1775-48d6-9dd2-ad1b55199455" />

<img width="663" height="395" alt="Image" src="https://github.com/user-attachments/assets/5c8fce5d-db3d-4604-8dc8-f28fcc4006a5" />

---

## Task 3: Enforce Least Privilege with a Scoped Policy

```Bash
# 3.1 Create a read-only user
aws $EP iam create-user --user-name Analyst_AlifAqlan

# 3.2 Attach a scoped, read-only policy (S3 read-only)
aws $EP iam attach-user-policy --user-name Analyst_AlifAqlan \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
  
# 3.3 List what the user can do
aws $EP iam list-attached-user-policies --user-name Analyst_AlifAqlan
```

**Evidence:**

<img width="598" height="192" alt="Image" src="https://github.com/user-attachments/assets/63d92430-7d3a-49ca-a166-c953b94ecd19" />

<img width="592" height="52" alt="Image" src="https://github.com/user-attachments/assets/cc4e131c-218e-4ae0-a40b-3d299963885b" />

<img width="660" height="172" alt="Image" src="https://github.com/user-attachments/assets/952017cd-ab6b-4159-b153-b0c44a379cb6" />

---

## Task 4: Credential Hygiene & Access Keys

```bash
# 4.1 Create an access key for the Analyst
aws $EP iam create-access-key --user-name Analyst_Alifaqlan

# 4.2 List access keys (note the AccessKeyId and status)
aws $EP iam list-access-keys --user-name Analyst_Alifaqlan

# 4.3 Rotate: deactivate the old key
aws $EP iam update-access-key --user-name Analyst_AlifAqlan \
  --access-key-id <PASTE_KEY_ID> --status Inactive
```

**Evidence:**

<img width="632" height="188" alt="Image" src="https://github.com/user-attachments/assets/0a733628-82c2-4dbb-9d1d-093d4b283ecc" />

<img width="547" height="211" alt="Image" src="https://github.com/user-attachments/assets/8bb54a5f-8d21-4391-987f-143a230f730c" />

<img width="583" height="52" alt="Image" src="https://github.com/user-attachments/assets/10e80073-16c9-402f-b27a-c6348c8e875d" />

---

# Session B (Week 2): Enforced Access Control with Kubernetes RBAC

LocalStack demonstrates the *mechanics* of IAM, but doesn't enforce anything end-to-end. Kubernetes RBAC does — this session shows access control actually blocking an unauthorised action.

## Environment Setup: Create a Local Kubernetes Cluster

```bash
kind create cluster --name ccse-lab1
kubectl cluster-info --context kind-ccse-lab1
kubectl get nodes
```

**Evidence:**

<img width="662" height="577" alt="Image" src="https://github.com/user-attachments/assets/d6810050-4335-42d8-973f-969b560a9d7d" />

---

## Task 5: Separate Environments with Namespaces

```bash
kubectl create namespace dev
kubectl create namespace prod
kubectl get namespaces
```

---

## Task 6: Define a Role and Bind It (Least Privilege)

```
# 6.1 Create a service account to represent a developer
kubectl create serviceaccount dev-user -n dev

# 6.2 Create a Role that allows only get/list/watch on pods in dev
kubectl create role pod-reader -n dev \
  --verb=get,list,watch --resource=pods

# 6.3 Bind the Role to the service account
kubectl create rolebinding dev-user-binding -n dev \
  --role=pod-reader --serviceaccount=dev:dev-user
```

**Evidence:**

<img width="432" height="52" alt="Image" src="https://github.com/user-attachments/assets/6f1258ad-8d74-4768-894c-ee8980d24795" />

<img width="447" height="67" alt="Image" src="https://github.com/user-attachments/assets/971e8933-8054-4c7d-9a60-2fcc2b3338a3" />

<img width="570" height="72" alt="Image" src="https://github.com/user-attachments/assets/5e9abbdf-2864-4300-90bd-988cbf73f630" />

---

## Task 7: Test That Access Control Works

```bash
SA="system:serviceaccount:dev:dev-user"

# Should be YES - reading pods in dev is allowed
kubectl auth can-i list pods -n dev --as=$SA

# Should be NO - deleting pods is not granted
kubectl auth can-i delete pods -n dev --as=$SA

# Should be NO - the role does not extend to prod
kubectl auth can-i list pods -n prod --as=$SA
```

**Evidence — (YES / NO / NO):**

<img width="438" height="191" alt="Image" src="https://github.com/user-attachments/assets/2064efa6-386c-4666-82d5-db26550bb821" />

---

# Deliverables & Assessment Answers

## 1. Short-Answer Questions

**Q1. Why is attaching policies to groups better than attaching them directly to users?**

Managing permissions through groups is much cleaner and easier to scale. Instead of updating permissions for individual users one by one whenever someone joins, changes roles, or leaves, you just add or remove them from a group. It prevents "permission drift" (where users accumulate random extra access over time) and makes auditing way simpler, as security admins only need to review group policies rather than inspect every single user account.

**Q2. What is the difference between an IAM User and an IAM Role?**

An IAM User is a permanent identity tied to a specific person or app, using long-term credentials like passwords or static access keys. An IAM Role, on the other hand, has no permanent credentials. Instead, trusted entities (like users, applications, or AWS services) temporarily "assume" the role to receive short-lived, auto-expiring credentials for that specific session, making it a much safer option for workloads.

**Q3. Explain least privilege using the Analyst account, and how it reduces blast radius if compromised.**

Least privilege means giving an identity only the exact permissions required to do its job and nothing more. In this lab, the `Analyst_Aisy` account was restricted to `AmazonS3ReadOnlyAccess`. If these credentials were ever leaked or stolen, the attacker could only view S3 files—they couldn't delete data, modify settings, spin up servers, or touch other AWS services. This limits the "blast radius" by containing the damage to S3 read access instead of compromising the whole cloud infrastructure.

**Q4. In Kubernetes, what is the difference between a Role and a RoleBinding?**

A **Role** defines *what* actions are allowed (such as `get`, `list`, or `watch` on pods) within a specific namespace. A **RoleBinding** defines *who* gets those permissions by connecting that Role to a subject (like a `ServiceAccount` or `User`). A Role on its own grants access to nobody until a RoleBinding explicitly attaches it to an identity.

**Q5. Why did the developer service account fail to access prod, and which security principle does that demonstrate?**

The `dev-user` service account couldn't access `prod` because both the `pod-reader` Role and `dev-user-binding` were created exclusively inside the `dev` namespace. Kubernetes RBAC uses a "default-deny" model—if there is no explicit rule granting access in `prod`, the request is blocked. This demonstrates **least privilege** and **namespace isolation**, ensuring that a dev account cannot reach or mess with production resources.

---

## 2. Authentication vs. Authorization Analysis

In all three `kubectl auth can-i` tests, the cluster identified the caller as `system:serviceaccount:dev:dev-user`. This means **Authentication** succeeded in every attempt—Kubernetes successfully validated the service account token and confirmed *who* was making the request.

* `list pods -n dev` ➔ **YES**: This request matched the `get`, `list`, and `watch` permissions explicitly granted by the `pod-reader` Role in the `dev` namespace.
* `delete pods -n dev` ➔ **NO**: The `delete` verb was never included in the Role's allowed actions, so the API server rejected the command under default-deny rules.
* `list pods -n prod` ➔ **NO**: The RoleBinding was scoped strictly to `dev`. Without a corresponding RoleBinding in `prod`, access is blocked by default.

In short: **Authentication** answers *"Who are you?"* (which passed every time), while **Authorization** answers *"What are you allowed to do?"*—and that is the security layer that blocked the unauthorized delete and cross-namespace actions.

---

## 3. Verification Command Output

```yaml
# kubectl get rolebinding dev-user-binding -n dev -o yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-user-binding
  namespace: dev
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: pod-reader
subjects:
- kind: ServiceAccount
  name: dev-user
  namespace: dev
```

---


## Security Best-Practices Checklist

- [x] Root user is not used for daily tasks (a dedicated admin identity exists)

- [x] Permissions are granted via groups/roles, not directly to individual users

- [x] At least one least-privilege (read-only) identity was created and tested

- [x] Access keys were listed and a rotation (deactivate) was demonstrated

- [x] Kubernetes RBAC blocks an unauthorized action (delete / cross-namespace)

---

## Cleanup & Teardown

```bash
# Remove the Kubernetes cluster
kind delete cluster --name ccse-lab1

# Stop and remove LocalStack
docker stop localstack && docker rm localstack
```

---

## References

1. Course lectures — Week 1 (Fundamentals), Week 2 (Security Architecture), Week 5 (Access Control), Week 7 (Identity Management)

2. LocalStack documentation — `docs.localstack.cloud`

3. Kubernetes RBAC — `kubernetes.io/docs/reference/access-authn-authz/rbac`

4. CSA Security Guidance v5 — Domain on Identity & Access Management





























