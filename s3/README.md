# Lab 2 — S3: Storage and Object Management

**Time:** ~45 minutes  
**Cost:** Free tier eligible (5GB storage, 20,000 GET requests/month)  
**Goal:** Understand S3 — the storage layer that underpins almost every AI system on AWS.

---

## 🧠 What is S3?

S3 (Simple Storage Service) is AWS's object storage. In the AI world it's everywhere — storing training datasets, model weights, pipeline artifacts, logs, and inference outputs. Every AI company uses it heavily.

---

## 🔧 Part 1 — Create your first bucket

### Step 1 — Open S3 console
1. Search for **S3** in the AWS console
2. Click **Create bucket**

### Step 2 — Configure the bucket
- **Bucket name:** `ricky-aws-labs-[random-number]` (must be globally unique)
- **Region:** US East (N. Virginia) — us-east-1
- **Block Public Access:** Leave all checked (keep it private for now)
- Click **Create bucket**

---

## 🔧 Part 2 — Upload and manage files

### Step 1 — Upload a file manually
1. Click your bucket
2. Click **Upload**
3. Upload any file from your computer (a text file works fine)
4. Click **Upload**

### Step 2 — Access the file
1. Click the file you uploaded
2. Click **Open** — notice it works because you're authenticated
3. Copy the **Object URL** and paste it in a private/incognito browser — you should get an Access Denied error
4. That's S3 permissions working correctly

---

## 🔧 Part 3 — AWS CLI with S3

This is how you'll actually use S3 in pipelines and scripts.

First configure the CLI if you haven't:
```bash
aws configure
# Enter your Access Key ID
# Enter your Secret Access Key
# Region: us-east-1
# Output format: json
```

Now try these commands:

```bash
# List your buckets
aws s3 ls

# List contents of your bucket
aws s3 ls s3://your-bucket-name/

# Upload a file
aws s3 cp myfile.txt s3://your-bucket-name/

# Download a file
aws s3 cp s3://your-bucket-name/myfile.txt ./downloaded-file.txt

# Sync a whole folder
aws s3 sync ./local-folder s3://your-bucket-name/folder/

# Delete a file
aws s3 rm s3://your-bucket-name/myfile.txt
```

---

## 🔧 Part 4 — S3 for AI use cases

This is where it gets relevant to your goals. Create a folder structure that mimics a real AI pipeline:

```bash
# Create a simulated ML project structure
aws s3api put-object --bucket your-bucket-name --key datasets/
aws s3api put-object --bucket your-bucket-name --key models/
aws s3api put-object --bucket your-bucket-name --key artifacts/
aws s3api put-object --bucket your-bucket-name --key logs/

# Upload a fake dataset (create a sample CSV first)
echo "id,input,output\n1,hello,world\n2,foo,bar" > sample_data.csv
aws s3 cp sample_data.csv s3://your-bucket-name/datasets/

# List the structure
aws s3 ls s3://your-bucket-name/ --recursive
```

This is exactly the kind of structure AI teams use to organise training data and model artifacts.

---

## 🔧 Part 5 — Bucket versioning

Enable versioning so you can recover deleted or overwritten files — critical for model artifact management:

```bash
aws s3api put-bucket-versioning \
  --bucket your-bucket-name \
  --versioning-configuration Status=Enabled
```

Now upload the same file twice with different content and check versions:
```bash
echo "version 1" > test.txt
aws s3 cp test.txt s3://your-bucket-name/

echo "version 2" > test.txt
aws s3 cp test.txt s3://your-bucket-name/

# List versions
aws s3api list-object-versions --bucket your-bucket-name
```

---

## 🔧 Part 6 — Lifecycle policies

Automatically move old files to cheaper storage — important when storing large model training artifacts:

1. Go to your bucket → **Management** tab
2. Click **Create lifecycle rule**
3. Name it `archive-old-artifacts`
4. Scope: Apply to all objects
5. Add transition: Move to **S3 Glacier** after 90 days
6. Save

This is real cost management that AI companies use for large datasets.

---

## 🧹 Cleanup

```bash
# Empty the bucket first
aws s3 rm s3://your-bucket-name/ --recursive

# Then delete it
aws s3api delete-bucket --bucket your-bucket-name
```

---

## ✅ What you learned

- How to create and manage S3 buckets
- How to use the AWS CLI for S3 operations
- How S3 is structured for AI/ML pipelines
- Versioning and lifecycle management

---

## 💭 Reflection questions

1. How would you structure S3 for a team of 10 ML engineers working on multiple models?
2. What's the difference between S3 Standard, S3-IA, and Glacier? When would you use each?
3. How would you secure an S3 bucket so only specific services can access it?

---

## 📝 My notes

> Add your own observations, what broke, what surprised you, and what you learned here.
