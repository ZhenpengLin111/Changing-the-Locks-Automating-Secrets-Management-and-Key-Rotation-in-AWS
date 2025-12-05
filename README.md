# Changing the Locks: Automating Secrets Management and Key Rotation in AWS

This project demonstrates a production-grade architecture for **securely managing and rotating database credentials** using:

- **AWS Secrets Manager**

- **AWS Lambda (custom rotation function)**

- **AWS RDS MySQL**

- **EC2 Instance (application test client)**

- **VPC Interface Endpoint for Secrets Manager**

- **CloudFormation for full automation**

The solution eliminates hardcoded credentials, enforces least privilege, and automates strong password rotation for both admin and application users.

---

## ⭐ Features

- 🔐 Automatic rotation of MySQL admin and app user passwords

- 🏗️ Fully automated deployment using AWS CloudFormation

- 🔒 Private subnets only for RDS + Lambda (no internet access)

- 🔌 Secure service access via Secrets Manager VPC Endpoint

- 🧩 Dual-secret architecture (Admin + App User)

- 🛡️ IAM least-privilege model

- 🧪 EC2 test client to validate end-to-end connectivity

- 📜 CloudWatch logging for rotation tracing and debugging

---

## 📐 Architecture Overview
![Alt text](architecture.png)

---

## 🔧 Components
### AWS Secrets Manager

Stores two secrets:

- `DBAdminSecret` – admin MySQL credentials

- `AppUserSecret` – limited-privilege application user

Both are automatically rotated every **30 days**.

---

### AWS Lambda – Rotation Function

Implements all 4 AWS rotation steps:

1. `createSecret`

2. `setSecret`

3. `testSecret`

4. `finishSecret`

Includes PyMySQL for RDS access.

Lambda responsibilities:

- Rotate **admin** credentials using `SET PASSWORD`

- Rotate **app user** credentials using `ALTER USER`
(requires first retrieving the admin secret)

---

### Amazon RDS MySQL

- Hosted in a **private subnet**

- No public access

- Only EC2 and Lambda can connect on port **3306**

---

### EC2 Test Instance

- Verifies secret retrieval and DB login

- Has IAM permission to read **only AppUserSecret**

- Contains a Python test script for validation

---

### VPC Endpoint – Secrets Manager

Allows EC2 + Lambda to access Secrets Manager **entirely inside the VPC**.

---

## 📦 Deployment
### 1️⃣ Create PyMySQL Lambda Layer (Required)

Run locally or in Cloud9:
```bash
mkdir -p python/lib/python3.11/site-packages
cd python/lib/python3.11/site-packages

pip install pymysql -t .

cd ../../../..
zip -r pymysql-layer.zip python/

aws lambda publish-layer-version \
    --layer-name pymysql-layer \
    --description "PyMySQL library for Python 3.11" \
    --zip-file fileb://pymysql-layer.zip \
    --compatible-runtimes python3.11 \
    --region us-east-1
```
Copy the output **Layer ARN** and paste into the CloudFormation template.

---

### 2️⃣ Deploy the CloudFormation Stack

Upload `cloudsec_rotation.yaml` to CloudFormation and deploy.

This creates:

- VPC + Subnets

- RDS MySQL instance

- Secrets

- Lambda rotation function

- EC2 instance

- VPC endpoint

- IAM policies

---

### 3️⃣ Create the App User in MySQL

After the DB is deployed:
```sql
CREATE USER 'app_user'@'%' IDENTIFIED BY 'TempApp_UserPassword123!';
GRANT SELECT, INSERT, UPDATE, DELETE ON cloudsecdb.* TO 'app_user'@'%';
FLUSH PRIVILEGES;
```

Lambda will rotate this password automatically.

---

### 4️⃣ Test the System (from EC2)

Retrieve password:
```bash
python3 test_db_connection.py
```

Expected result:

- Secret retrieved successfully

- MySQL login succeeds

- Shows end-to-end rotation working

---

## 🔐 Security Model
### IAM Roles
Role	Access
Lambda Rotation Role	Read + rotate both secrets
EC2 Instance Role	Read only AppUserSecret
Admin Secret	Never exposed to EC2
AppUserSecret	Least-privilege access for application

---

### Security Groups

- Lambda → RDS: port **3306** only

- EC2 → RDS: port **3306** only

- EC2 → Secrets Manager via VPC Endpoint: port **443**

- Lambda → Secrets Manager via VPC Endpoint: port **443**

- EC2 (SSH remote login only)

---

## 🧪 End-to-End Validation Flow
```pgsql
EC2 → IAM Authorization
    → VPC Routing + Security Groups
    → Secrets Manager VPC Endpoint
    → Secrets Manager (GetSecretValue)
    → EC2 receives password
    → MySQL login on port 3306
```

If this succeeds, the rotation pipeline is fully functional.