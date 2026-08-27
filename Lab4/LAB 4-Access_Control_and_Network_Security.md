# Lab 4: Access Control & Network Security

## Course Information

- **Course Name:** IKB42603 Cloud Computing Security Essentials
- **Instructor:** MADAM ADANI
- **Student Name:** MUHAMMAD ALIFF AQLAN BIN MUHAMMAD ARIFF
- **Topic:** Authentication, authorization, MFA, network segmentation, firewall rules and container hardening
- **Environment:** Kali Linux, Docker, Nginx, Kubernetes, Kind, kubectl, oathtool, iptables and Trivy
- **Date:** 27 August 2026

## Lab Objectives

The objectives of this lab are:

- To distinguish between authentication and authorization.
- To protect a web service using HTTP Basic Authentication.
- To generate and validate a TOTP code as a second authentication factor.
- To enforce least-privilege access using Kubernetes RBAC.
- To implement network segmentation between the web, application and database tiers.
- To configure default-deny firewall rules with explicit allowed traffic.
- To harden a container using non-root access, a read-only filesystem and restricted Linux capabilities.
- To scan a container image for known vulnerabilities using Trivy.

## Learning Outcomes

After completing this lab, I was able to:

- Configure a password-protected Nginx web service.
- Test unauthenticated and authenticated access using HTTP status codes.
- Generate and validate a six-digit TOTP code.
- Create a Kubernetes service account, role and role binding.
- Test allowed and denied actions using Kubernetes RBAC.
- Separate containers using frontend and backend Docker networks.
- Verify that the web tier could not directly reach the database tier.
- Configure a default-deny firewall policy using iptables.
- Run a hardened container as a non-root user with a read-only filesystem.
- Drop unnecessary Linux capabilities and prevent privilege escalation.
- Scan a container image for high and critical vulnerabilities using Trivy.

## Environment

| Component | Details |
|---|---|
| Operating System | Kali Linux |
| Container Platform | Docker |
| Web Server | Nginx |
| Authentication Method | HTTP Basic Authentication |
| MFA Tool | oathtool |
| MFA Method | TOTP |
| Kubernetes Cluster Tool | Kind |
| Kubernetes Version | v1.30.0 |
| Kubernetes Command-Line Tool | kubectl |
| Access Control | Kubernetes RBAC |
| Network Segmentation | Docker networks |
| Database Service | Redis |
| Firewall Tool | iptables |
| Hardened Image | nginxinc/nginx-unprivileged |
| Vulnerability Scanner | Trivy |
| Working Directory | `~/Lab4` |

## Lab Summary

In this lab, an Nginx web service was protected using HTTP Basic Authentication, and a TOTP code was generated and validated as a second authentication factor. Kubernetes RBAC was configured to allow a developer service account to list pods while denying permission to create deployments and delete pods. Docker networks were used to separate the web, application and database tiers, preventing the web container from directly reaching the database. A default-deny firewall was also configured to allow only required traffic. Finally, a container was hardened using a non-root user, a read-only filesystem, dropped capabilities and the no-new-privileges option. The `nginx:alpine` image was then scanned using Trivy to identify known high and critical vulnerabilities.

# Session A (Week 7) — Authentication & Authorization

## Task 1: Authentication - Password-Protected Service

1. Create a password file for the user 'student' with the password 'Password!':

```
docker run --rm httpd:alpine htpasswd -nbB student 'P@ssword!' > htpasswd.txt
```

2. Create an Nginx configuration file (default.conf) to apply basic auth protection:

```
cat > default.conf <<'EOF'
server {
    listen 80;
    location / {
        auth_basic "Restricted";
        auth_basic_user_file /etc/nginx/.htpasswd;
        return 200 'Authenticated OK\n';
    }
}
EOF
```

3. Run the Nginx container using the configuration file and the password:

```
docker run --rm -d --name authsvc -p 8080:80 \
  -v "$(pwd)/default.conf:/etc/nginx/conf.d/default.conf" \
  -v "$(pwd)/htpasswd.txt:/etc/nginx/.htpasswd" nginx
```

4. Test the connection using curl (without credentials, it returns a 401 error; with credentials, it returns "200 Authenticated OK").

```
curl -o /dev/null -w 'no-creds: %{http_code}\n' http://localhost:8080
curl -s -u student:'P@ssword!' http://localhost:8080
```

The 401 response confirmed that access without credentials was rejected. The 200 response confirmed that the correct username and password successfully authenticated the user.
Evidence :

<img width="887" height="218" alt="Image" src="https://github.com/user-attachments/assets/245c77d4-25b6-4ce7-81a1-7aa353bec856" />


## Task 2: Add a Second Factor (MFA / TOTP)

1. Generate a shared secret code (in base32 format) and produce the current 6-digit OTP code:
   
```
SECRET=$(head -c20 /dev/urandom | base32)
echo "Enrol this secret in an authenticator app: $SECRET"
oathtool --totp -b "$SECRET"
```

2. Verify the entered code (it will ask you to enter the 6-digit code for verification):

```
read -p 'Enter the 6-digit code: ' CODE
[ "$CODE" = "$(oathtool --totp -b "$SECRET")" ] && echo 'MFA OK' || echo 'MFA FAILED'
```

This confirmed that the correct TOTP code was entered and successfully validated as a second authentication factor.

Evidence :
<img width="902" height="460" alt="Image" src="https://github.com/user-attachments/assets/62cf916b-cdf7-45e3-9c29-b7846ff9a757" />



## Task 3: Authorization Using Kubernetes RBAC

1. Create a Kubernetes cluster

```
kind create cluster --name ccse-lab4
```

2. Create a namespace and service account for developers.

```
kubectl create namespace app
kubectl create serviceaccount dev -n app
```

3. Create a role (read-only access to pods) and bind it to the developer.

```
kubectl create role dev-role --verb=get,list --resource=pods -n app
kubectl create rolebinding dev-rb --role=dev-role --serviceaccount=app:dev -n app
```

4. Authorization check

```
SA=system:serviceaccount:app:dev
kubectl auth can-i list pods -n app --as=$SA
kubectl auth can-i create deployments -n app --as=$SA
kubectl auth can-i delete pods -n app --as=$SA
```

By preventing the dev service account from creating deployments or deleting pods while allowing it to list them, Kubernetes RBAC successfully demonstrated least-privilege access enforcement.

Evidence :

<img width="738" height="306" alt="Image" src="https://github.com/user-attachments/assets/992dc402-a138-49f3-892f-645a230466ef" />


# Session B (Week 8) — Network Security & Hardening

## Task 4 — Network Segmentation (Three-Tier)

1. Create two segmented networks

```
docker network create frontend-net
docker network create backend-net
```

2. Run the database container (only on backend-net), the app (on both networks), and the web (on frontend-net)

```
docker run -d --name db --network backend-net redis:alpine
docker run -d --name app --network backend-net nginx
docker network connect frontend-net app
docker run -d --name web --network frontend-net nginx
```

3. Test the connection from the web to the database (must FAIL / be BLOCKED due to different networks).

```
docker exec web sh -c 'apk add -q curl; curl -m 3 db:6379 || echo BLOCKED'
```

4. Test the connection from the app to the DB (must be SUCCESSFUL / REACHABLE as they share the backend-net)

```
docker exec app sh -c 'apk add -q curl; nc -z -w3 db 6379 && echo REACHABLE'
```

The web container couldn't reach the database directly since they weren't on the same Docker network. However, the application container was able to connect to the database via backend-net.

Evidence:

<img width="827" height="305" alt="Image" src="https://github.com/user-attachments/assets/a575d8e0-8aa4-4398-bb0b-945258a123aa" />

## Task 5: Firewall Rules (Default-Deny).

1. Apply host-level firewall rules that permit only the ports you need. This mirrors cloud security groups.

```
docker run --rm --cap-add=NET_ADMIN alpine sh -c '\
apk add -q iptables; \
iptables -P INPUT DROP; \
iptables -A INPUT -p tcp --dport 443 -j ACCEPT; \
iptables -A INPUT -i lo -j ACCEPT; \
iptables -L INPUT -v -n'
```

This exemplified the default-deny model, where network traffic is automatically dropped unless an explicit allow rule is defined.

Evidence:

<img width="833" height="205" alt="Image" src="https://github.com/user-attachments/assets/0c812f68-1dc1-4883-8f7a-8f8a66af9e06" />


## Task 6: Container / Host Hardening & Vulnerability Scanning.

1. Reduce the attack surface. Build a minimal, non-root, capability-dropped, read-only container and scan it
   Run hardened container services.
   
```
docker run -d --name hardened \
  --user 1000:1000 \
  --read-only \
  --cap-drop ALL \
  --security-opt no-new-privileges \
  --tmpfs /tmp \
  nginxinc/nginx-unprivileged
```

2. Check the security configuration settings for the container.

```
docker inspect hardened --format 'User={{.Config.User}} ReadOnly={{.HostConfig.ReadonlyRootfs}}'
```

3. Scan images to detect HIGH and CRITICAL level vulnerabilities using Trivy

```
docker run --rm aquasec/trivy image --severity HIGH,CRITICAL nginx:alpine | head -20
```

Container hardening inspection and Trivy vulnerability scan summary results.

Evidence:

<img width="1003" height="137" alt="Image" src="https://github.com/user-attachments/assets/550ebd1d-986d-449a-af5a-97736228ac12" />

<img width="1322" height="332" alt="Image" src="https://github.com/user-attachments/assets/00a7b72d-5095-4154-9273-45c105ae439d" />

## Hardening Measures and Reduced Attack Surfaces

| Hardening measure     | Attack or risk reduced                                                                                      |
| --------------------- | ----------------------------------------------------------------------------------------------------------- |
| Non-root user         | Reduces the impact of a container compromise because the service does not have root privileges.             |
| Read-only filesystem  | Prevents attackers from modifying system files or installing malicious files in the container filesystem.   |
| Drop all capabilities | Removes unnecessary Linux privileges that could be abused for privilege escalation or system-level actions. |


## Short-answer questions

### 1. Explain the difference between authentication and authorization using Tasks 1 and 3.
- Authentication refers to verifying who you are, which is demonstrated in Task 1 by using HTTP Basic Authentication to check valid user credentials before permitting entry to the web service. Meanwhile, authorization decides what you may do, which is shown in Task 3 where Kubernetes RBAC restricts an authenticated developer service account to only read pods (list pods) while denying unauthorized actions like creating deployments or deleting pods.

### 2. Why is MFA so effective, and which attacks does it defeat?
- MFA is highly effective because it requires multiple verification factors from different security classes, combining something you know (such as a password) with something you have (such as a time-based TOTP code). Because of this, it successfully defeats the majority of standard credential-based attacks like password brute-forcing, credential stuffing, and keylogging, since a compromised password alone is useless without the second factor.

### 3. How does network segmentation limit the damage of a compromised web server?
- Network segmentation physically or logically isolates tiers—such as the frontend, backend, and database—into separate networks. This design effectively prevents lateral movement, meaning if an external attacker successfully compromises the internet-facing web tier, they still cannot establish a direct path to reach the secure database tier.

### 4. What does a default-deny firewall policy achieve, and how does it relate to cloud security groups?
- A default-deny firewall policy establishes the strict security principle of least privilege by blocking all network traffic by default, requiring explicit permission rules for any necessary ports. This model directly relates to cloud security groups, where nothing is allowed through unless it has been explicitly authorized.

### 5. List the hardening measures you applied and the attack surface each one removes.
- The core measures were running as a non-root user, mounting the root filesystem read-only, and dropping all Linux capabilities. These controls respectively reduce privilege after service compromise, block filesystem tampering and persistence, and remove access to privileged kernel operations. The container also used no-new-privileges to prevent privilege gains and a tmpfs mount for non-persistent temporary writes. Finally, an Alpine-based Nginx image was scanned with Trivy to identify known high- and critical-severity package vulnerabilities.

### Security best-practices checklist

- [x] Service requires authentication; unauthenticated requests are rejected.
- [x] MFA/second factor was generated and successfully validated.
- [x] RBAC enforces least privilege; unauthorized actions are denied.
- [x] Network segmentation prevents direct web-to-database communication.
- [x] Default-deny firewall uses explicit allow rules.
- [x] Container runs as non-root.
- [x] Container root filesystem is read-only.
- [x] Linux capabilities are dropped.
- [x] `no-new-privileges` is enabled.
- [x] Image was scanned for high- and critical-severity vulnerabilities.

## Cleanup and teardown

The lab resources were removed after evidence collection.

```
docker rm -f authsvc db app web hardened 2>/dev/null
docker network rm frontend-net backend-net 2>/dev/null
kind delete cluster --name ccse-lab4
```

The cleanup output shows removal of the remaining containers, both custom networks, and the `ccse-lab4` kind cluster. `authsvc` had already been stopped earlier and was automatically removed because it was started with `--rm`.

Evidence:

<img width="491" height="200" alt="Image" src="https://github.com/user-attachments/assets/0897239a-dd55-4d62-ad50-06c54ee4086a" />


### Evidence completeness and issues found

All required deliverables from the guide are present:

| Guide requirement | Evidence | Status |
|---|---|---:|
| HTTP `401` without credentials and `200` with valid credentials | Task 1 authentication screenshot | Complete |
| Valid TOTP produces `MFA OK` | Task 2 screenshot | Complete |
| Three `kubectl auth can-i` results | Task 3 RBAC screenshot | Complete |
| `web -> db: BLOCKED` and `app -> db: REACHABLE` | Task 4 results screenshot | Complete |
| Default-deny iptables ruleset | Task 5 screenshot | Complete |
| Hardened-container inspect output | Task 6 inspect screenshot | Complete |
| Trivy scan summary | Task 6 Trivy summary screenshot | Complete |
| Role-binding YAML verification | Task 3 RBAC screenshot | Complete |
| Dropped-capabilities verification | Task 6 inspect screenshot | Complete |


## Conclusion

In conclusion, all tasks in Lab 4 were successfully completed by applying a comprehensive defense-in-depth security approach covering applications, networks, and hosts. Through Tasks 1 to 3, identity authentication and role-based access control (RBAC) were successfully tested to ensure that only authorized users possess the correct permissions. Additionally, implementing multi-factor authentication (MFA/TOTP) further strengthened account security against credential-based attacks.

































































