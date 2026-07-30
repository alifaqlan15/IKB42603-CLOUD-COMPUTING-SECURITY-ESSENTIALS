# Lab 0: Environment Setup Report

| Item | Details |
|---|---|
| Course | IKB42603 Cloud Computing Security Essentials |
| Lab | Lab 0 — Environment Setup |
| Reference | IKB42603_Lab0_Environment_Setup_Cheatsheet.pdf |
| Platform | Windows (Git Bash) |

---

## 1. Objective

The purpose of Lab 0 was to prepare the local lab environment so the later labs could run without a real AWS account. The guide required the installation and verification of Docker, AWS CLI v2, kind, kubectl, and LocalStack, followed by a local Kubernetes cluster test.

---

## 2. Tools & Environment Summary

| Tool | Purpose | Verified Version / Status |
|---|---|---|
| **Docker** | Runs containers and LocalStack | Installed & Running |
| **AWS CLI v2** | Sends AWS commands to LocalStack | `aws-cli/2.x` |
| **kind** | Runs local Kubernetes cluster inside Docker | `v0.23.0` |
| **kubectl** | Controls the Kubernetes cluster | `v1.36.x` |
| **LocalStack** | Local AWS simulator | Running on port 4566 |

---

## 3. Step-by-step Execution

### Step 1 — Install and verify Docker
```bash
docker --version
docker run --rm hello-world
```bash
docker --version
docker run --rm hello-world
```

Evidence: [Docker verification] 
<img width="406" height="35" alt="s1" src="https://github.com/user-attachments/assets/bbed457d-2dff-439c-805d-dd16e3a50989" />

---

### Step 2 — Install and verify AWS CLI v2

The AWS CLI installation was checked to verify that the system can communicate with and manage simulated cloud services:

Evidence: [aws --version]
<img width="456" height="36" alt="s1" src="https://github.com/user-attachments/assets/2fe2fc4f-eb31-4f01-983c-07e2b0c22a22" />

The output confirms AWS CLI version is installed and running on the local environment.

---

### Step 3 — Kind

The kind utility was checked to ensure the system is capable of spinning up local Kubernetes clusters using Docker containers:

Evidence: [kind --version]
<img width="386" height="31" alt="s1" src="https://github.com/user-attachments/assets/eb0165c4-4345-421d-89ce-040f65cb6d42" />

The output confirms kind is properly installed and available in the executable path.

---

### Step 4 — Kubectl

The kubectl CLI tool was checked to ensure it can interact with Kubernetes cluster control planes:

Evidence: [kubectl version --client]

<img width="477" height="56" alt="s1" src="https://github.com/user-attachments/assets/3d6a79b6-bd10-4394-ad71-60a916e57187" />

The command returned the client version details for kubectl.

---

### Step 5 — Verify Helper Tools

The guide listed OpenSSL and Oathtool as required helper tools for cryptographic operations and MFA/OTP verification in upcoming security labs:

Evidence: [openssl version] 

<img width="547" height="37" alt="s1" src="https://github.com/user-attachments/assets/5adff266-0253-45b9-88dd-403f4e0f6ab6" />

The output confirms that the cryptographic and authentication helper tools are available in the local environment path.

---

### Step 6 — Start and verify LocalStack

The guide instructed the user to start LocalStack to simulate AWS cloud services locally:

Evidence: [docker run -d --name localstack -p 4566:4566 localstack/localstack:2.3.2
curl http://localhost:4566/_localstack/health]

<img width="657" height="240" alt="s1" src="https://github.com/user-attachments/assets/736512c4-dca6-427b-ae3d-343d03d416c2" />

Observed result:

The LocalStack container was pulled and started successfully.

The health check endpoint was reachable on localhost port 4566.

---

### Step 7 — LocalStack & AWS Identity Check

LocalStack was started via Docker to simulate AWS cloud services locally. The connection was verified using the AWS CLI caller identity check:

Evidence: [aws --endpoint-url=http://localhost:4566 sts get-caller-identity]

<img width="675" height="126" alt="s1" src="https://github.com/user-attachments/assets/915bf00f-e650-4a66-aec7-b01f6c67bc91" />

The output confirms successful connection to LocalStack with the dummy AWS account identity.

---

### Step 8 — Kubernetes Cluster Setup

A local Kubernetes cluster named  (ccse) was created and inspected to ensure the node state is ready:

Evidence: [kind create cluster --name ccse kubectl get nodes]

<img width="666" height="107" alt="s1" src="https://github.com/user-attachments/assets/3d9728c3-1d9c-4541-9b8d-091b9571e10b" />

The output confirms the ccse-control-plane node is in the Ready status.

---

### 4.Pre-lab Verification Checklist
- [x] Docker version was verified and hello-world container ran successfully.

- [x] AWS CLI v2 version was verified.

- [x] kind version was verified.

- [x] kubectl client version was verified.

- [x] OpenSSL helper tool was verified.

- [x] LocalStack started and health check was reachable locally.

- [x] AWS CLI successfully connected to LocalStack with dummy identity.

- [x] Kubernetes cluster (ccse) was created and control-plane node was Ready.

---

### 5.Conclusion
The Lab 0 environment setup was successfully completed according to the course guide. All required core tools (Docker, AWS CLI v2, kind, kubectl, OpenSSL, LocalStack, and Kubernetes) were verified. The local environment is fully operational and ready for the subsequent cloud security labs.












