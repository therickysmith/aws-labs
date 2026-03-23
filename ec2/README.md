# Lab 1 — EC2: Compute Fundamentals

**Time:** ~1 hour  
**Cost:** Free tier eligible  
**Goal:** Launch, connect to, and manage an EC2 instance. Understand how AWS compute works before touching anything more complex.

---

## 🧠 What is EC2?

EC2 (Elastic Compute Cloud) is AWS's virtual machine service. Think of it like spinning up a VM in Azure — same concept, different interface. AI companies use EC2 (especially GPU instances) to run model training and inference workloads.

---

## 🔧 Part 1 — Launch your first EC2 instance

### Step 1 — Open the EC2 console
1. Log into AWS console at console.aws.amazon.com
2. Search for **EC2** in the top search bar
3. Click **Launch Instance**

### Step 2 — Configure the instance
Fill in the following:
- **Name:** `my-first-ec2`
- **AMI (OS):** Amazon Linux 2023 (free tier eligible)
- **Instance type:** `t2.micro` (free tier eligible)
- **Key pair:** Click "Create new key pair"
  - Name it `my-aws-key`
  - Type: RSA
  - Format: `.pem`
  - **Download and save this file somewhere safe — you can't download it again**

### Step 3 — Network settings
- Leave defaults for now
- Make sure **Allow SSH traffic** is checked

### Step 4 — Launch
Click **Launch Instance**. Takes about 60 seconds to start.

---

## 🔧 Part 2 — Connect to your instance

### Step 1 — Get the public IP
1. Go to EC2 → Instances
2. Select your instance
3. Copy the **Public IPv4 address**

### Step 2 — SSH in
On Mac/Linux:
```bash
chmod 400 ~/Downloads/my-aws-key.pem
ssh -i ~/Downloads/my-aws-key.pem ec2-user@YOUR_PUBLIC_IP
```

On Windows (PowerShell):
```powershell
ssh -i C:\Users\YourName\Downloads\my-aws-key.pem ec2-user@YOUR_PUBLIC_IP
```

You should see a terminal prompt like `[ec2-user@ip-xxx ~]$` — you're in. 🎉

---

## 🔧 Part 3 — Install something and run it

Install nginx and start it:
```bash
sudo dnf install nginx -y
sudo systemctl start nginx
sudo systemctl status nginx
```

Now go to your EC2 security group and add an inbound rule:
- Type: HTTP
- Port: 80
- Source: Anywhere (0.0.0.0/0)

Open your browser and go to `http://YOUR_PUBLIC_IP` — you should see the nginx welcome page. You just deployed a web server on AWS.

---

## 🔧 Part 4 — Stop and start (don't terminate yet)

1. Go to EC2 → Instances
2. Select your instance → Instance State → **Stop**
3. Notice the public IP changes when you restart it
4. This is why you use **Elastic IPs** for persistent addresses in production

---

## 🔧 Part 5 — User Data (automated startup scripts)

This is the DevOps magic — scripts that run automatically when an instance launches.

1. Launch a new instance (same settings as before)
2. Expand **Advanced Details** at the bottom
3. In the **User Data** field paste:

```bash
#!/bin/bash
dnf install nginx -y
systemctl start nginx
systemctl enable nginx
echo "<h1>Deployed by Ricky on AWS EC2</h1>" > /usr/share/nginx/html/index.html
```

4. Launch the instance
5. Wait 2 minutes, then open the public IP in your browser

That's automated infrastructure provisioning — the same concept behind every deployment pipeline you'll work with at an AI company.

---

## 🧹 Cleanup

**Important — stop or terminate instances when not using them to avoid charges.**

1. EC2 → Instances → Select instance → Instance State → **Terminate**

---

## ✅ What you learned

- How to launch and connect to an EC2 instance
- How security groups control network access
- How User Data scripts automate instance configuration
- Why IPs change on stop/start and how to handle it

---

## 💭 Reflection questions

Think through these — they're the kind of things you'll be asked in interviews:

1. How is EC2 different from running a container? When would you use one vs the other?
2. What would you use instead of User Data scripts for more complex configuration?
3. How would you make this setup production-ready? What's missing?

---

## 📝 My notes

> Add your own observations, what broke, what surprised you, and what you learned here.
