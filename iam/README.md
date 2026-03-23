# Lab 3 — IAM: Security, Roles and Permissions

**Time:** ~1 hour  
**Cost:** Free (IAM has no cost)  
**Goal:** Understand IAM — the security layer that controls who and what can access AWS resources. Critical for production AI systems.

---

## 🧠 What is IAM?

IAM (Identity and Access Management) controls access to everything in AWS. In AI companies, getting IAM wrong is how data leaks happen and how bills explode. Understanding IAM deeply is a real differentiator for a DevOps engineer.

---

## 🔧 Part 1 — Explore IAM users and groups

### Step 1 — Create an IAM user
1. Search for **IAM** in the AWS console
2. Click **Users** → **Create user**
3. Name: `devops-user`
4. Check **Provide user access to the AWS Management Console**
5. Set a password
6. Click through to create

### Step 2 — Create a group with permissions
1. Click **User groups** → **Create group**
2. Name: `devops-team`
3. Attach policy: search for and add **AmazonS3ReadOnlyAccess**
4. Create group
5. Add `devops-user` to the group

### Step 3 — Test the permissions
Log in as `devops-user` using the console sign-in URL (shown in IAM → Users → devops-user → Security credentials).

Try to:
- List S3 buckets ✅ (should work)
- Create an EC2 instance ❌ (should be denied)

That's least-privilege access working correctly.

---

## 🔧 Part 2 — IAM Roles (the important part)

Roles are how AWS services talk to each other securely. This is used constantly in AI pipelines — allowing EKS to access S3, allowing SageMaker to write to S3, allowing Lambda to call APIs.

### Step 1 — Create a role for EC2 to access S3
1. IAM → **Roles** → **Create role**
2. Trusted entity: **AWS service**
3. Use case: **EC2**
4. Attach policy: **AmazonS3FullAccess**
5. Name: `ec2-s3-access-role`
6. Create role

### Step 2 — Attach the role to an EC2 instance
1. Go to EC2 → launch a new instance (or use your existing one)
2. Under **Advanced Details** → **IAM instance profile** → select `ec2-s3-access-role`
3. Launch the instance

### Step 3 — Test it
SSH into the instance and try:
```bash
# This should work without configuring any credentials
aws s3 ls

# Upload a file
echo "hello from ec2" > test.txt
aws s3 cp test.txt s3://your-bucket-name/
```

No access keys needed — the role handles authentication automatically. This is how production systems work.

---

## 🔧 Part 3 — IAM Policies in depth

### Step 1 — Create a custom policy
1. IAM → **Policies** → **Create policy**
2. Click **JSON** tab and paste:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::your-bucket-name",
        "arn:aws:s3:::your-bucket-name/*"
      ]
    }
  ]
}
```

3. Name it `specific-bucket-access`
4. Create policy

This policy grants access to only one specific bucket — exactly what you'd do in production to isolate AI model storage.

### Step 2 — Attach to a role
Create a new role, attach this custom policy instead of the broad S3 full access. This is least-privilege principle in action.

---

## 🔧 Part 4 — IAM for AI pipelines

Create the role structure you'd use in a real AI deployment:

```bash
# Create a policy for ML pipeline access
cat > ml-pipeline-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::your-bucket-name/models/*",
        "arn:aws:s3:::your-bucket-name/datasets/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "*"
    }
  ]
}
EOF

aws iam create-policy \
  --policy-name ml-pipeline-policy \
  --policy-document file://ml-pipeline-policy.json
```

---

## ✅ What you learned

- How IAM users, groups, and roles work
- The difference between users (humans) and roles (services)
- How to write custom IAM policies in JSON
- How roles enable secure service-to-service communication without credentials

---

## 💭 Reflection questions

1. Why is using IAM roles better than using access keys for EC2 instances?
2. What is the principle of least privilege and why does it matter for AI systems?
3. If a SageMaker training job needs to read from S3 and write results to another S3 bucket, how would you set up the IAM role?

---

## 📝 My notes

> Add your own observations, what broke, what surprised you, and what you learned here.
