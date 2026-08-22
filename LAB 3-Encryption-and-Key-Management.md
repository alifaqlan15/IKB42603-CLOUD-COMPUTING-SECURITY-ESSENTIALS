# Lab 3: Encryption & Key Management

## 1. Lab Overview

This lab demonstrates encryption and key management techniques in a cloud-native environment.

### Covered Topics
* Symmetric encryption using **AES**
* Asymmetric encryption and digital signatures using **RSA**
* Encryption in transit using **TLS/HTTPS**
* **AWS KMS** operations using **LocalStack**
* Data key generation and envelope encryption
* Tenant-specific key management
* Cryptographic erasure
* **SHA-256** integrity verification and hash chaining

---

## 2. Learning Outcomes

By completing this lab, the following concepts and practical skills were demonstrated:

- [x] Symmetric encryption and decryption using AES.
- [x] RSA key-pair generation and digital signature verification.
- [x] TLS encryption configuration for securing data in transit.
- [x] KMS key creation and cryptographic operations.
- [x] Data key generation and implementation of envelope encryption.
- [x] Key separation and isolation between multiple tenants.
- [x] Cryptographic erasure and its direct effect on encrypted data accessibility.
- [x] SHA-256 hashing algorithms and data integrity verification.

---

## 3. Environment & Configuration

### Tools Used
* **Kali Linux** (Operating System)
* **OpenSSL** (Cryptographic toolkit)
* **Docker** (Container runtime)
* **Nginx** (Web server for TLS configuration)
* **AWS CLI** (Command-line interface for AWS services)
* **LocalStack** (Local AWS cloud emulator)

### LocalStack Endpoint
```text
http://localhost:4566
```

### Working Directory
```text
cd ~/IKB42603-CLOUD-COMPUTING-SECURITY-ESSENTIALS/Lab3_Encryption_and_Key_Management
```
---

# Session A (Week 5) — Encryption Fundamentals

## 4. Task 1 — AES Symmetric Encryption

In this task, AES-256 symmetric cryptography was used to protect sensitive data at rest, applying a single shared secret key for both the encryption and decryption processes.

Creating the plaintext record and encrypting it with OpenSSL (AES-256-CBC using PBKDF2 for key derivation):

### Create the initial record
```
echo 'Patient: Alif Aqlan, Diagnosis: confidential' > record.txt
```

### Encrypt the file using AES-256-CBC
```
openssl enc -aes-256-cbc -pbkdf2 -salt -in record.txt -out record.enc
```

### Display unreadable raw encrypted content
```
cat record.enc
```

### Decrypt the file back to plaintext
```
openssl enc -d -aes-256-cbc -pbkdf2 -in record.enc -out record.dec.txt
```

### Compare files to confirm an exact match
```
diff record.txt record.dec.txt && echo 'MATCH: decryption successful'
```

### The output shows a successful match, meaning the file got decrypted back to its exact original form without any errors using the correct key

Evindence :

<img width="642" height="126" alt="Image" src="https://github.com/user-attachments/assets/b74a26c6-974f-4394-91b4-ccce6ac98e1b" />

<img width="685" height="171" alt="Image" src="https://github.com/user-attachments/assets/1d0d711b-bac3-4339-a1a4-e5de19efacdf" />


## 5. Task 2: Asymmetric Encryption & Digital Signatures

In this part, an RSA key pair was generated to test asymmetric cryptography. Unlike symmetric keys, the public key is used for encryption while the private key handles decryption, and signatures are created using the private key to prove origin and integrity.

Generating the 2048-bit RSA private and public key files:


### Generate an RSA key pair
```
openssl genrsa -out private.pem 2048
openssl rsa -in private.pem -pubout -out public.pem
```

### Encrypting the record using the public key and decrypting it back with the private key
```
openssl pkeyutl -encrypt -pubin -inkey public.pem -in record.txt -out record.rsa
openssl pkeyutl -decrypt -inkey private.pem -in record.rsa -out record.rsa.txt
```

### Signing the document with the private key and verifying its authenticity using the public key
```
openssl dgst -sha256 -sign private.pem -out record.sig record.txt
openssl dgst -sha256 -verify public.pem -signature record.sig record.txt
```

## The output shows a successful verification, meaning the digital signature is valid and matches the original file without any tampering

Evidence :

<img width="671" height="53" alt="Image" src="https://github.com/user-attachments/assets/52a32d19-318e-48fa-9d1f-b7efdd599c6a" />

---

## 6. Task 3: Encryption in Transit (TLS)

In this task, a self-signed SSL certificate was generated to test encryption in transit. A local Nginx container was set up to serve the file over HTTPS, demonstrating how TLS protects network traffic from being intercepted or read in clear text.

Generating the self-signed certificate and running the HTTPS container:


### Generate a self-signed certificate valid for 7 days

```
openssl req -x509 -newkey rsa:2048 -keyout key.pem -out cert.pem -days 7 -nodes -subj '/CN=localhost'
```

### Serve the file over HTTPS on port 8443 using an Nginx container

```
docker run --rm -d --name tls -p 8443:443 \
  -v $(pwd)/cert.pem:/etc/nginx/cert.pem \
  -v $(pwd)/key.pem:/etc/nginx/key.pem \
  -v $(pwd)/record.txt:/usr/share/nginx/html/record.txt nginx
```

### Accessing the file securely using curl over TLS (using -k to accept the self-signed certificate)

```
curl -k https://localhost:8443/record.txt
```

### The secure connection successfully retrieved the file contents, proving that the channel is encrypted and protected from eavesdropping in transit.

Evidence :

<img width="426" height="171" alt="Image" src="https://github.com/user-attachments/assets/c347bb20-ff13-4e3a-94a8-d3027e3a863c" />

# Session B (Week 6) — Key Management, Envelope Encryption &
Erasure

## Task 4 — AWS KMS Using LocalStack

### 4.1 Start LocalStack

The LocalStack container was started to provide an emulated AWS cloud environment locally:

```bash
docker run -d --name localstack -p 4566:4566 localstack/localstack:2.3.2
```

## The endpoint environment variable was configured to point AWS CLI to LocalStack:

```
EP='--endpoint-url=http://localhost:4566'
```
LocalStack provides a local AWS-compatible environment for testing KMS operations.

Evidence :

<img width="283" height="69" alt="Image" src="https://github.com/user-attachments/assets/8e52699b-733e-4a95-ade7-78187e6ce8a4" />


## Create a Master Key and Encrypt Data
A Customer Master Key (CMK) was created for Tenant A

```
aws $EP kms create-key --description 'CCSE tenant-A master key'
```
Evidence:

<img width="846" height="379" alt="Image" src="https://github.com/user-attachments/assets/4e928d9c-7a9c-4f7a-a1b6-a83826ab6aff" />

## After capturing the generated KeyId into a variable (KEY_A), a small piece of text was encrypted directly using KMS:

```
KEY_A=<PASTE_KEYID_A>
aws $EP kms encrypt --key-id $KEY_A --plaintext "$(echo -n 'hello' | base64)" --query CiphertextBlob --output text
```

Evidence :

<img width="906" height="167" alt="Image" src="https://github.com/user-attachments/assets/2133a118-4b35-41b3-a103-401337cfa68a" />

## Existing keys in the local KMS service were verified using

```
aws $EP kms list-keys
```

Evidence:

<img width="894" height="155" alt="Image" src="https://github.com/user-attachments/assets/dd05c723-4e66-4b97-8079-9dd409baea5c" />




## 8. Task 5 — Data Key and Envelope Encryption

### 8.1 Generate a Data Key

A 256-bit AES data key was generated using KMS:

```bash
aws $EP kms generate-data-key \
--key-id $KEY_A \
--key-spec AES_256 \
--query '[Plaintext,CiphertextBlob]' \
--output text
```
The generated plaintext data key was saved for the encryption operation while the encrypted version was retained as the protected data key.

Evidence:

<img width="905" height="151" alt="Image" src="https://github.com/user-attachments/assets/29ce6945-8166-47df-8709-666827544a02" />

### 8.2 Store the Data Key Files
The generated data key material was stored in files for the envelope encryption demonstration:

### Save column 1 as datakey.b64 and column 2 as datakey.enc
```
echo "$DATA_KEY_RAW" | awk '{print $1}' > datakey.b64
echo "$DATA_KEY_RAW" | awk '{print $2}' > datakey.enc
```

### Encrypt the file locally with the plaintext data key
```
base64 -d datakey.b64 > datakey.bin
openssl enc -aes-256-cbc -pbkdf2 -in record.txt -out record.env.enc -pass file:./datakey.bin
```

### Clean up plaintext material and keep only wrapped key
```
rm datakey.bin datakey.b64
```
Evidence :

<img width="838" height="223" alt="Image" src="https://github.com/user-attachments/assets/ad99ca16-f370-48a8-82ab-f810511ee24e" />

## 9. Task 6 — Tenant Key Separation and Cryptographic Erasure

### 9.1 Create a Tenant-B KMS Key

A separate KMS key was created for tenant-B:

```bash
aws $EP kms create-key --description 'CCSE tenant-B master key'
```

The resulting Key ID was recorded and used as the tenant-B key.

Using separate keys provides logical cryptographic separation between tenants.

Evidence :

<img width="842" height="375" alt="Image" src="https://github.com/user-attachments/assets/e0777971-018f-4c9a-a21c-dbcac036486c" />


### 9.2 Cryptographic Erasure via Key Deletion 

To demonstrate cryptographic erasure, tenant-A's master key was disabled and scheduled for deletion:

### Schedule deletion of tenant A's key
```
aws $EP kms schedule-key-deletion --key-id$KEY_A --pending-window-in-days 7
```

### Disable the key immediately to simulate erasure
```
aws $EP kms disable-key --key-id$KEY_A
```

### An attempt was then made to decrypt the wrapped data key (datakey.enc) using the disabled/erased key, which resulted in a failure:

```
aws $EP kms decrypt --ciphertext-blob fileb://datakey.enc 2>&1 | head -3
```

Evidence :

<img width="904" height="496" alt="Image" src="https://github.com/user-attachments/assets/6e38ec17-7277-4608-b171-a09387992105" />

## 10. Task 7 — Integrity & Tamper-Evidence

### 10.1 File Fingerprinting & Tamper Detection

The baseline cryptographic fingerprint of the record file was generated using SHA-256:

```
sha256sum record.txt
```

### When a copy of the file was modified (tampered.txt), the resulting hash changed entirely, immediately revealing the modification:

```
cp record.txt tampered.txt; echo 'x' >> tampered.txt
sha256sum record.txt tampered.txt
```

Evidence :

<img width="710" height="142" alt="Image" src="https://github.com/user-attachments/assets/72207c8e-4b9c-4c86-85ab-0eddba1eab55" />


### 10.2 Tamper-Evident Hash Chain Log

A simple hash chain was constructed where each log entry cryptographically incorporates the hash of the preceding entry:

```
PREV=0
for line in 'login ok' 'file read' 'export data'; do \
  PREV=$(echo -n "$PREV$line" | sha256sum | cut -d' ' -f1); \
  echo "$line \vert{}$PREV"; \
done
```

Evidence :

<img width="756" height="156" alt="Image" src="https://github.com/user-attachments/assets/7cae7071-370d-4e6e-88ea-2557a22b410f" />


## 11. Final Verification

The RSA signature was verified again using:

```bash
openssl dgst -sha256 -verify public.pem -signature record.sig record.txt
```

The verification result was:

```
Verified OK
```
## 12. Short-Answer Questions

### Q1. Compare symmetric and asymmetric encryption: speed, key distribution, and typical use.
* **Speed:** Symmetric encryption is much faster because it uses a single key for both locking and unlocking data, whereas asymmetric is slower due to heavy mathematical processing with public-private key pairs.
* **Key Distribution:** Symmetric is harder to share securely because both sides need the exact same secret key beforehand. Asymmetric solves this by letting anyone lock data with a public key, while only the owner can unlock it with their private key.
* **Typical Use:** Symmetric is used for bulk data storage or heavy file encryption, while asymmetric is typically used for secure handshakes (like TLS), digital signatures, and exchanging keys safely.

### Q2. Why is key management described as the weakest link, not the algorithm?
Modern cryptographic algorithms like AES-256 or RSA-2048 are mathematically solid and practically impossible to break by brute force today. However, human or system errors usually happen around the keys—like hardcoding keys in source code, leaving them exposed in public repositories, or poor access control. If someone gets hold of the key, the strongest algorithm in the world won't protect the data.

### Q3. Explain envelope encryption and why only the master key needs hardware-grade protection.
Envelope encryption means we encrypt our actual data using a local data key, and then we encrypt (wrap) that data key using a master key stored safely in KMS. Only the master key needs heavy, hardware-grade protection (like an HSM) because it guards all the child keys. Since data keys are only used temporarily or for specific files, protecting just the master key saves performance while keeping everything secure.

### Q4. How does cryptographic erasure achieve provable deletion where overwriting cannot (in the cloud)?
In cloud storage, traditional file overwriting isn't guaranteed to wipe data completely because of how cloud drives replicate data across multiple physical disks and clusters. Cryptographic erasure bypasses this by destroying or disabling the encryption master key. Even if the encrypted data fragments stay sitting on old cloud storage sectors forever, nobody can ever read them because the key to decrypt them is gone for good.

### Q5. How does a hash chain make a log tamper-evident?
A hash chain links each log entry to the one right before it by combining the previous entry's hash with the new data. If an attacker tries to change or delete an old log entry in the middle of the chain, the hash for that block changes, which completely breaks all the subsequent hashes that follow it. This makes tampering instantly noticeable during an audit or verification check.


## 13. Security Best-Practices Checklist

[x] AES was used for symmetric encryption.

[x] RSA was used for asymmetric cryptography and digital signatures.

[x] SHA-256 was used for integrity verification.

[x] TLS was used to protect data in transit.

[x] KMS was used for encryption key management.

[x] Data keys were used for envelope encryption.

[x] Separate KMS keys were demonstrated for different tenants.

[x] Plaintext key material was removed after use.

[x] Cryptographic erasure was demonstrated.

[x] Hash chaining was used to detect changes to recorded data.

## 14. Conclusion

Overall, this lab gave a really practical view of how data security and key management actually work in cloud environments. Working through everything hands on from standard file encryption and digital signatures to setting up a local TLS connection helped clear up how confidentiality and integrity are protected during transit. 

Using LocalStack to simulate AWS KMS made it much easier to understand how cloud services handle master keys, data keys, and tenant separation without needing a real cloud account. The exercises on envelope encryption and cryptographic erasure also showed how effectively data can be securely wiped just by destroying its decryption key. 

To wrap it up, the hashing and hash-chain tasks proved just how important cryptographic checks are for spotting any unauthorized changes to records, showing that solid security relies on proper protection both at rest and in transit.



















































































