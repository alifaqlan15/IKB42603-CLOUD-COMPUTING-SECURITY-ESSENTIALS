# Lab 6 Report: Object Storage Security & the Data Security Lifecycle

---

**Course:** IKB42603 Cloud Computing Security Essentials  

**Student Name:** MUHAMMAD ALIFF AQLAN BINMUHAMMAD ARIFF 

**Lecturer:** MADAM ADANI 

---

## Lab Learning Outcomes

At the end of this lab, I was able to:

1. ✅ Provision object storage (S3), classify the data we store, and understand why this storage style has a different security model compared to block or file storage.
2. ✅ Recreate the classic cloud leak scenario—a publicly readable bucket—and fix it using Block Public Access and a least-privilege bucket policy.
3. ✅ Differentiate between identity-based policies (IAM) and resource-based policies (bucket policy), and predict what happens when both policies disagree.
4. ✅ Enforce encryption at rest using SSE-KMS so files are automatically encrypted upon upload.
5. ✅ Use presigned URLs to share file access for a limited time window while understanding the risks.
6. ✅ Apply versioning, set up lifecycle and retention rules, observe leftover old data (data remanence), and achieve provable deletion through cryptographic erasure

---

# Session A (Week 11) - Object Storage & the Exposure Problem

## Setup & Configuration

### Starting LocalStack

Start clean with IAM enforcement enabled

```
docker rm -f localstack 2>/dev/null
docker run -d --name localstack -p 4566:4566 \
  -e LOCALSTACK_AUTH_TOKEN=$LOCALSTACK_AUTH_TOKEN \
  -e ENFORCE_IAM=1 \
  localstack/localstack-pro:latest
```

Point the CLI at LocalStack

```
export EP='--endpoint-url=http://localhost:4566'
aws configure set aws_access_key_id test
aws configure set aws_secret_access_key test
aws configure set region us-east-1
aws $EP sts get-caller-identity
```

---

## Task 1: Classify the Data Before You Store It

In this section, I set up a dedicated S3 bucket for the hospital records system and uploaded three types of files with different sensitivity levels, adding a classification tag to each one.

### Commands Executed

```
export BUCKET=miit-patient-records-$RANDOM
echo $BUCKET

aws --endpoint-url=http://localhost:4566 s3api create-bucket --bucket $BUCKET

echo 'Ward visiting hours 10am-8pm' > public-notice.txt
echo 'Staff duty schedule, week 12' > internal-roster.txt
echo 'Patient: Ahmad bin Ali, Diagnosis: confidential' > confidential-record.txt

aws --endpoint-url=http://localhost:4566 s3api put-object --bucket $BUCKET --key public/notice.txt --body public-notice.txt --tagging 'classification=public'
aws --endpoint-url=http://localhost:4566 s3api put-object --bucket $BUCKET --key internal/roster.txt --body internal-roster.txt --tagging 'classification=internal'
aws --endpoint-url=http://localhost:4566 s3api put-object --bucket $BUCKET --key confidential/record.txt --body confidential-record.txt --tagging 'classification=confidential'

aws --endpoint-url=http://localhost:4566 s3api list-objects-v2 --bucket $BUCKET --query 'Contents[].[Key,Size]' --output table
aws --endpoint-url=http://localhost:4566 s3api get-object-tagging --bucket $BUCKET --key confidential/record.txt
```

| Classification | Who may read it | Impact if leaked | Control you will apply |
| :--- | :--- | :--- | :--- |
| **public** | Anyone (Public / Anonymous users) | No negative impact since the information is meant for everyone (e.g., hospital visiting hours). | Block Public Access & Least-Privilege Policy |
| **internal** | Hospital staff / Internal organization members only | Could disrupt internal operations or duty schedules if leaked to external parties. | Least-Privilege Bucket Policy (Scoped to internal/*) |
| **confidential** | Authorized personnel / Medical officers only | Serious patient privacy violation, risk of legal action, and exposure of personal health records (e.g., PDPA/GDPR breach). | SSE-KMS Encryption, Deny Policy, & Versioning/Lifecycle |

Evidence :

<img width="610" height="267" alt="Image" src="https://github.com/user-attachments/assets/db89c02a-f006-4d2d-b318-5b5e7edae460" />

---

## Task 2 — Reproducing the Archetypal Breach

In this section, I simulated a real-world cloud security breach scenario to demonstrate how an unhardened S3 bucket allows unauthorized users to extract sensitive data.

### Command used:

Attempting to download the confidential patient record

```
aws --endpoint-url=http://localhost:4566 s3api get-object \
    --bucket $BUCKET \
    --key confidential/record.txt \
    leaked-record.txt
```
Inspecting the contents of the leaked file

```
cat leaked-record.txt
```


### Terminal Output Evidence:

```
{
    "AcceptRanges": "bytes",
    "LastModified": "2026-09-08T18:29:20+00:00",
    "ContentLength": 48,
    "ETag": "\"9a86d9c8a68fe26ab3f63cd85c116a7f\"",
    "ContentType": "binary/octet-stream",
    "ServerSideEncryption": "AES256",
    "Metadata": {},
    "TagCount": 1
}

Patient: Ahmad bin Ali, Diagnosis: confidential
```

Findings:

- Vulnerability Confirmed: The S3 bucket allowed the object confidential/record.txt to be fetched directly without blocking public/unauthorized read requests.
- Impact: The sensitive medical diagnosis of the patient (Patient: Ahmad bin Ali, Diagnosis: confidential) was retrieved and read using cat leaked-record.txt. This represents an archetypal S3 data leak resulting from missing access restrictions and unconfigured Block Public Access settings.

Evidence:

<img width="553" height="325" alt="Image" src="https://github.com/user-attachments/assets/d697179f-e625-48fb-82ec-92b00082332d" />

---

## Task 3 — Remediate with Block Public Access

In this section, I removed the dangerous public policy and applied the Block Public Access configuration as a preventative guardrail. Afterward, I established a secure least-privilege resource policy restricted strictly to my own account and the internal prefix.

### Command Used:

Remove the vulnerable public policy
```
aws $EP s3api delete-bucket-policy --bucket $BUCKET
```

Apply the Block Public Access configuration guardrail
```
aws $EP s3api put-public-access-block --bucket $BUCKET \
    --public-access-block-configuration \
    BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true

aws $EP s3api get-public-access-block --bucket $BUCKET
```

Try to re-introduce the public policy (should be blocked/guarded)
```
aws $EP s3api put-bucket-policy --bucket $BUCKET --policy file://public-policy.json
```

Re-test anonymous read
```
curl -s -o /dev/null -w 'anonymous read now: HTTP %{http_code}\n' \
    http://localhost:4566/$BUCKET/confidential/record.txt
```

Apply the proper least-privilege policy for your own account (internal prefix only)
```
cat > least-privilege-policy.json <<JSON
{
"Version": "2012-10-17",
"Statement": [{
"Sid": "AccountReadInternalOnly",
"Effect": "Allow",
"Principal": {"AWS": "arn:aws:iam::000000000000:root"},
"Action": "s3:GetObject",
"Resource": "arn:aws:s3:::$BUCKET/internal/*"
}]
}
JSON

aws $EP s3api put-bucket-policy --bucket $BUCKET --policy file://least-privilege-policy.json
aws $EP s3api get-bucket-policy --bucket $BUCKET --query Policy --output text
```

### Observation & Analysis:

- Preventative Guardrail: Enabling Block Public Access overrides any future accidental bucket policies or ACLs that attempt to make the storage public.
- Least-Privilege Enforcement: The new resource policy (least-privilege-policy.json) restricts read actions exclusively to the root account (000000000000) and limits it specifically to the internal/* prefix, effectively protecting confidential files from unauthorized exposure.

Evidence :

<img width="581" height="180" alt="Image" src="https://github.com/user-attachments/assets/3637d731-39c3-42d7-a418-d14671ae57c2" />

--- 

## Task 4 — Identity Policy vs Resource Policy.

In this task, we will examine what happens when an identity-based policy (attached to the IAM user) conflicts with a resource-based policy (attached to the S3 bucket). We will create a user named DataAnalyst who is granted permission to read all files, but the bucket policy will enforce an explicit deny on the confidential/ folder.


### Command Used:

Create a new IAM user named DataAnalyst
```
aws $EP iam create-user --user-name DataAnalyst
```

Create the IAM policy file for the analyst (Allow reading everything)
```
cat > analyst-iam.json <<'JSON'
{
"Version": "2012-10-17",
"Statement": [{
"Effect": "Allow",
"Action": ["s3:GetObject", "s3:ListBucket"],
"Resource": "*"
}]
}
JSON
```

Attach the policy to the DataAnalyst user
```
aws $EP iam put-user-policy --user-name DataAnalyst \
    --policy-name S3ReadAll-policy-document file://analyst-iam.json
```

Get the Access Keys for the analyst profile
```
aws $EP iam create-access-key --user-name DataAnalyst \
    --query 'AccessKey.[AccessKeyId,SecretAccessKey]' --output text
```

### Note: After you run the 4th command above, it will output two key values (Access Key ID and Secret Access Key). Copy both values and insert them into the command below (replace PASTE_KEY_ID_HERE and PASTE_SECRET_HERE without removing the quotes):

Insert the newly retrieved keys into these variables
```
ANALYST_KEY_ID='PASTE_KEY_ID_HERE'
ANALYST_SECRET='PASTE_SECRET_HERE'
```

Configure the AWS profile named 'analyst'
```
aws configure --profile analyst set aws_access_key_id "$ANALYST_KEY_ID"
aws configure --profile analyst set aws_secret_access_key "$ANALYST_SECRET"
aws configure --profile analyst set region us-east-1
```

### Next, lets set up the bucket resource policy that allows internal access but explicitly denies access to confidential files:

Create the deny-confidential.json file
```
cat > deny-confidential.json <<JSON
{
"Version": "2012-10-17",
"Statement": [
{
"Sid": "AllowAnalystInternal",
"Effect": "Allow",
"Principal": {"AWS": "arn:aws:iam::000000000000:user/DataAnalyst"},
"Action": "s3:GetObject",
"Resource": "arn:aws:s3:::$BUCKET/internal/*"
},
{
"Sid": "DenyAnalystConfidential",
"Effect": "Deny",
"Principal": {"AWS": "arn:aws:iam::000000000000:user/DataAnalyst"},
"Action": "s3:*",
"Resource": "arn:aws:s3:::$BUCKET/confidential/*"
}
]
}
JSON
```

Upload the bucket policy
```
aws $EP s3api put-bucket-policy --bucket $BUCKET --policy file://deny-confidential.json
```

### Test both access permissions:

Test 1: Read internal file (Should SUCCEED / ALLOWED)
```
AWS_PROFILE=analyst aws $EP s3api get-object \
    --bucket $BUCKET --key internal/roster.txt analyst-internal.txt && echo "internal: ALLOWED"
```

Test 2: Read confidential file (Should FAIL / DENIED by Explicit Deny)
```
AWS_PROFILE=analyst aws $EP s3api get-object \
    --bucket $BUCKET --key confidential/record.txt analyst-conf.txt || echo "confidential: DENIED"
```

Evidence :

Test 1 
<img width="772" height="272" alt="Image" src="https://github.com/user-attachments/assets/36623be6-182f-40fb-a9e5-4e2fb387a779" />

Test 2
<img width="753" height="245" alt="Image" src="https://github.com/user-attachments/assets/acb0e726-b93e-4283-b418-1f5247bd5184" />

--- 

# Session B (Week 12) -  Protecting, Retaining and Retiring Data


## Task 5 — Default Encryption at Rest (SSE-KMS).

In this task, we will create a dedicated customer-managed KMS key and configure it as the default encryption setting for our S3 bucket. This ensures that any future object uploaded without explicit encryption flags will automatically be encrypted by the bucket itself.

### Command Used:

Create a dedicated KMS key for the bucket
```
export KEY_ID=$(aws $EP kms create-key \
    --description 'IKB42603 Lab6 patient records bucket key' \
    --query 'KeyMetadata.KeyId' --output text)
echo $KEY_ID
```

Create the encryption configuration JSON file
```
cat > encryption.json <<JSON
{
"Rules": [{
"ApplyServerSideEncryptionByDefault": {
"SSEAlgorithm": "aws:kms",
"KMSMasterKeyID": "$KEY_ID"
},
"BucketKeyEnabled": true
}]
}
JSON
```

Apply the default encryption configuration to the bucket
```
aws $EP s3api put-bucket-encryption --bucket $BUCKET \
    --server-side-encryption-configuration file://encryption.json

aws $EP s3api get-bucket-encryption --bucket $BUCKET
```

Upload a new file with NO encryption flags to test default encryption
```
aws $EP s3api put-object --bucket $BUCKET \
    --key confidential/record-v2.txt \
    --body confidential-record.txt
```

Verify the object's encryption properties
```
aws $EP s3api head-object --bucket $BUCKET \
    --key confidential/record-v2.txt \
    --query '[ServerSideEncryption,SSEKMSKeyId,BucketKeyEnabled]' --output text
```

Evidence :

<img width="780" height="312" alt="Image" src="https://github.com/user-attachments/assets/75c7c74c-ba54-4a1e-83c7-7c049a4c9fee" />

--- 

## Task 6 — Delegated Access and the Condition Key Trap.

In this task, we will create a time-bounded presigned URL to share a file securely without sharing AWS credentials, and then test an over-strict TLS transport policy that intentionally locks us out in a plain HTTP LocalStack environment.

## Command Used:

Generate a presigned URL valid for 60 seconds
```
aws $EP s3 presign s3://$BUCKET/internal/roster.txt --expires-in 60
```

Save the generated URL into a variable (replace the string below with your actual generated URL)
```
URL='PASTE_PRESIGNED_URL_HERE'
```

Test downloading immediately using curl
```
curl -s -o /dev/null -w 'HTTP %{http_code}\n' "$URL"
```

The Condition-Key Trap: Create a secure transport policy denying unencrypted traffic
```
cat > secure-transport.json <<JSON
{
"Version": "2012-10-17",
"Statement": [{
"Sid": "DenyUnencryptedTransport",
"Effect": "Deny",
"Principal": "*",
"Action": "s3:*",
"Resource": ["arn:aws:s3:::$BUCKET", "arn:aws:s3:::$BUCKET/*"],
"Condition": {"Bool": {"aws:SecureTransport": "false"}}
}]
}
JSON
```

Apply the policy (this will cause you to lock yourself out on LocalStack HTTP endpoint)
```
aws $EP s3api put-bucket-policy --bucket $BUCKET --policy file://secure-transport.json
```

Test a standard command (expecting it to be refused)
```
aws $EP s3api list-objects-v2 --bucket $BUCKET
```

Recover before continuing
```
aws $EP s3api delete-bucket-policy --bucket $BUCKET
```

Evidence :

<img width="773" height="716" alt="Image" src="https://github.com/user-attachments/assets/6729fbf5-7f89-48a1-a8c1-40b6de7a0551" />

---

## Task 7 — Versioning, Delete Markers & Data Remanence.

In this task, we will enable bucket versioning to see how object deletions actually work in the cloud. When versioning is enabled, deleting an object does not permanently remove it; instead, it places a delete marker on top, allowing older versions (including sensitive data we tried to redact) to survive underneath.

## Command Used:

Enable bucket versioning
```
aws $EP s3api put-bucket-versioning --bucket $BUCKET \
    --versioning-configuration Status=Enabled
aws $EP s3api get-bucket-versioning --bucket $BUCKET
```

Create two new revisions of the same record with different diagnoses
```
echo 'Patient: Ahmad bin Ali, Diagnosis: hypertension' > rec-v2.txt
echo 'Patient: [REDACTED], Diagnosis: [REDACTED]' > rec-v3.txt
```

Upload revision 2 and revision 3
```
echo 'Patient: Ahmad bin Ali, Diagnosis: hypertension' > rec-v2.txt
echo 'Patient: [REDACTED], Diagnosis: [REDACTED]' > rec-v3.txt
```

List all object versions
```
aws $EP s3api list-object-versions --bucket $BUCKET \
    --prefix confidential/record.txt \
    --query 'Versions[].[VersionId,IsLatest,Size]' --output table
```

Delete the record (this places a delete marker instead of erasing it)
```
aws $EP s3api delete-object --bucket $BUCKET --key confidential/record.txt
```

Check the delete markers table
```
aws $EP s3api list-object-versions --bucket $BUCKET \
    --prefix confidential/record.txt \
    --query 'DeleteMarkers[].[VersionId,IsLatest]' --output table
```

Recover the original unredacted record (version-id null)
```
aws $EP s3api get-object --bucket $BUCKET --key confidential/record.txt \
    --version-id null recovered.txt
cat recovered.txt
```

Permanently delete the specific version to prevent data remanence
```
aws $EP s3api delete-object --bucket $BUCKET \
    --key confidential/record.txt --version-id null
```

Evidence :

<img width="697" height="956" alt="Image" src="https://github.com/user-attachments/assets/19ae2710-55e7-4d67-ba78-eae57aa150db" />

--- 

## Task 8 — Lifecycle, Retention & Cryptographic Erasure.

In this task, we will configure an automated lifecycle policy to handle version expiration and cleanup, and perform cryptographic erasure by disabling and scheduling the deletion of our KMS key, rendering all encrypted data unrecoverable noise.

## Command Used:

Create the lifecycle configuration JSON file
```
cat > lifecycle.json <<'JSON'
{
"Rules": [
{
"ID": "RetireConfidentialRecords",
"Filter": {"Prefix": "confidential/"},
"Status": "Enabled",
"Expiration": {"Days": 365},
"NoncurrentVersionExpiration": {"NoncurrentDays": 30}
},
{
"ID": "AbortIncompleteUploads",
"Filter": {"Prefix": ""},
"Status": "Enabled",
"AbortIncompleteMultipartUpload": {"DaysAfterInitiation": 7}
}
]
}
JSON
```

Apply and verify the lifecycle configuration
```
aws $EP s3api put-bucket-lifecycle-configuration --bucket $BUCKET \
    --lifecycle-configuration file://lifecycle.json

aws $EP s3api get-bucket-lifecycle-configuration --bucket $BUCKET \
    --query 'Rules[].[ID,Status]' --output table
```

Cryptographic Erasure: Disable and schedule deletion of the KMS key
```
aws $EP kms describe-key --key-id $KEY_ID \
    --query 'KeyMetadata.[KeyId,KeyState,Enabled]' --output text

aws $EP kms disable-key --key-id $KEY_ID

aws $EP kms schedule-key-deletion --key-id $KEY_ID --pending-window-in-days 7

aws $EP kms describe-key --key-id $KEY_ID \
    --query 'KeyMetadata.[KeyState,DeletionDate]' --output text
```

Finally, run the Verification Command block required at the end of the lab to generate the final security posture output

```
echo "=== IKB42603 Lab 6 verification: $BUCKET ==="

aws $EP s3api get-public-access-block --bucket $BUCKET \
    --query 'PublicAccessBlockConfiguration' --output text

aws $EP s3api get-bucket-versioning --bucket $BUCKET --output text

aws $EP s3api get-bucket-encryption --bucket $BUCKET \
    --query 'ServerSideEncryptionConfiguration.Rules[0].ApplyServerSideEncryptionByDefault.[SSEAlgorithm,KMSMasterKeyID]' \
    --output text

aws $EP s3api get-bucket-lifecycle-configuration --bucket $BUCKET \
    --query 'Rules[].[ID,Status]' --output text

aws $EP kms describe-key --key-id $KEY_ID \
    --query 'KeyMetadata.KeyState' --output text
```

Evidence:

<img width="707" height="902" alt="Image" src="https://github.com/user-attachments/assets/d3f3298e-ff68-4d8f-ac4c-48644ec9c4d6" />

<img width="753" height="492" alt="Image" src="https://github.com/user-attachments/assets/682b8975-af69-49df-8340-9b782b29e147" />


---

## Short Answer Questions

### Question 1
**Which single element of the Task 2 policy caused the exposure, and why is `Principal: "*"` more dangerous on a bucket policy than an over-broad IAM policy attached to one user?**
> The single element that caused the exposure was `"Principal": "*"`[cite: 1]. This wildcard makes the policy apply to everyone globally, including anonymous unauthenticated users over the public internet[cite: 1]. A broad IAM policy attached to a user still requires valid AWS credentials to authenticate, whereas a bucket policy with `Principal: "*"` bypasses authentication entirely, allowing anyone with the bucket URL to fetch data directly via standard HTTP tools like `curl` without any credentials[cite: 1].

### Question 2
**Explain the difference between an identity-based policy and a resource-based policy. In Task 4, which one decided each of the analyst's two requests?**
> Identity-based policies (like IAM user policies) are attached directly to an IAM user, group, or role to define what actions that identity can perform[cite: 1]. Resource-based policies (like S3 bucket policies) are attached directly to the resource (the bucket) to control who can access that specific resource and under what conditions[cite: 1]. In Task 4, the resource-based policy decided both requests: the `Allow` statement permitted access to the `internal/*` prefix, while the explicit `Deny` statement overrode the analyst's broad IAM permissions to block access to the `confidential/*` prefix[cite: 1].

### Question 3
**Block Public Access is described as a guardrail rather than a control. What is the difference, and why does the distinction matter for an organisation with many engineers?**
> A control (such as deleting an insecure bucket policy) fixes a specific misconfiguration reactively after it happens, whereas a guardrail is a preventive safety boundary that actively stops misconfigurations from being applied in the first place[cite: 1]. This distinction is critical in large organizations with multiple engineers because human errors happen; Block Public Access acts as an account or bucket-level safety net that overrides any future accidental bucket policies or ACLs, preventing colleagues from inadvertently exposing storage buckets to the public[cite: 1].

### Question 4
**Your bucket has default SSE-KMS encryption. Does that protect the confidential record from the analyst in Task 4? Explain precisely what server-side encryption does and does not defend against.**
> No, default SSE-KMS encryption does not protect the record from the analyst in Task 4[cite: 1]. Server-side encryption protects data at rest from physical theft or unauthorized inspection of the underlying storage media by automatically decrypting the data on the fly for authorized API requests[cite: 1]. It does not defend against authorization flaws or policy misconfigurations, because if an identity has valid API permissions (`s3:GetObject`) to access the file, AWS will successfully decrypt and serve it regardless of encryption[cite: 1].

### Question 5
**A patient invokes their right to erasure. Using your Task 7 evidence, explain why `delete-object` alone is not compliant, and describe two mechanisms that would make the deletion provable.**
> Running `delete-object` alone is not compliant because when bucket versioning is enabled, it only places a *delete marker* on top while preserving all prior unredacted versions underneath, resulting in object-level data remanence[cite: 1]. Two mechanisms that achieve provable deletion are: (1) permanently deleting every historical object version explicitly by specifying its unique `VersionId`[cite: 1], and (2) cryptographic erasure, which involves destroying the underlying KMS encryption key so that all existing ciphertext instantly becomes unrecoverable mathematical noise, regardless of how many versions or backups remain[cite: 1].

### Question 6
**You are the auditor in Week 11. Name three commands from this lab whose output you would collect as compliance evidence, and state which control each one evidences.**
> First, `aws s3api get-public-access-block` evidences that public exposure guardrails are actively enforced across all four protection flags[cite: 1]. Second, `aws s3api get-bucket-encryption` evidences compliance with data-at-rest encryption standards using customer-managed KMS keys[cite: 1]. Third, `aws s3api get-bucket-lifecycle-configuration` evidences that automated data retention and retirement rules are configured to handle records and version expirations properly[cite: 1].


---

## References

1. UniKL MIIT. (2026). *IKB42603 Cloud Computing Security Essentials: Lab Manual Lab 6 (Weeks 11-12) - Object Storage Security & the Data Security Lifecycle*. Universiti Kuala Lumpur.
2. Amazon Web Services. (2026). *Amazon S3 Security Best Practices*. Retrieved from https://docs.aws.amazon.com/AmazonS3/latest/userguide/security-best-practices.html
3. Amazon Web Services. (2026). *Using S3 Versioning and Lifecycle Policies*. Retrieved from https://docs.aws.amazon.com/AmazonS3/latest/userguide/Versioning.html
4. LocalStack. (2026). *LocalStack S3 Service Coverage and Limitations*. Retrieved from https://docs.localstack.cloud/references/coverage
5. Cloud Security Alliance (CSA). (2024). *CSA Security Guidance for Critical Areas of Focus in Cloud Computing v5 - Domain 5: Data Security*.
6. Malaysian Communications and Multimedia Commission (MCMC). (2021). *MCMC MTSFB TC G017:2021 - Information Security Requirements for Cloud Service Providers*.

























