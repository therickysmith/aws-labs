# Lab 4 — EKS: Kubernetes on AWS

**Time:** ~2-3 hours  
**Cost:** ~$0.30/hr while the cluster is running — EKS control plane ($0.10/hr) + 2x t3.medium nodes (~$0.10/hr each). Shut it down when done.  
**Goal:** Deploy and manage a Kubernetes cluster on AWS EKS. This is the core skill for AI infrastructure roles.

---

## 🧠 What is EKS?

EKS (Elastic Kubernetes Service) is AWS's managed Kubernetes service. You've already learned Kubernetes concepts with minikube — EKS is the production version. Anthropic, OpenAI, and most AI companies run their inference and deployment infrastructure on EKS or equivalent managed Kubernetes.

---

## 🛠️ Prerequisites

Complete Labs 1-3 first. Also install:

```bash
# Install eksctl (EKS management tool)
# Mac
brew tap weaveworks/tap
brew install weaveworks/tap/eksctl

# Linux
curl --silent --location \
  "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_Linux_amd64.tar.gz" \
  | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin

# Windows (PowerShell as admin)
winget install eksctl

# Verify
eksctl version

# Install kubectl if not already installed
kubectl version --client
```

---

## 🔧 Part 1 — Create your EKS cluster

### Step 1 — Create the cluster

```bash
eksctl create cluster \
  --name ai-labs-cluster \
  --region us-east-1 \
  --nodegroup-name standard-workers \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 3 \
  --managed
```

⚠️ This takes 15-20 minutes. Go make a coffee.

### Step 2 — Verify the cluster

```bash
# Check nodes are ready
kubectl get nodes

# Check all system pods are running
kubectl get pods --all-namespaces
```

You should see 2 nodes in Ready state.

---

## 🔧 Part 2 — Deploy an application

### Step 1 — Create a deployment manifest

```bash
cat > app-deployment.yaml << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-app
  labels:
    app: demo-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: demo-app
  template:
    metadata:
      labels:
        app: demo-app
    spec:
      containers:
      - name: demo-app
        image: nginx:latest
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
EOF
```

### Step 2 — Deploy it

```bash
kubectl apply -f app-deployment.yaml
kubectl get deployments
kubectl get pods
```

### Step 3 — Expose it with a Load Balancer

```bash
cat > app-service.yaml << EOF
apiVersion: v1
kind: Service
metadata:
  name: demo-app-service
spec:
  selector:
    app: demo-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: LoadBalancer
EOF

kubectl apply -f app-service.yaml

# Wait for external IP (takes 2-3 minutes)
kubectl get service demo-app-service --watch
```

When you see an external IP, open it in your browser. That's your app running on EKS with a real AWS load balancer in front of it.

---

## 🔧 Part 3 — Self healing and scaling

### Test self-healing

```bash
# Get pod names
kubectl get pods

# Delete one pod
kubectl delete pod <pod-name>

# Watch Kubernetes immediately replace it
kubectl get pods --watch
```

### Test scaling

```bash
# Scale up
kubectl scale deployment demo-app --replicas=5
kubectl get pods

# Scale down
kubectl scale deployment demo-app --replicas=2
kubectl get pods
```

### Set up autoscaling

```bash
kubectl autoscale deployment demo-app \
  --cpu-percent=50 \
  --min=2 \
  --max=10

kubectl get hpa
```

---

## 🔧 Part 4 — Connect EKS to S3 using IAM roles

This is a critical pattern for AI workloads — pods that need to access S3 for model weights or datasets.

### Step 1 — Enable OIDC provider

```bash
eksctl utils associate-iam-oidc-provider \
  --cluster ai-labs-cluster \
  --approve
```

### Step 2 — Create a service account with S3 access

```bash
eksctl create iamserviceaccount \
  --name s3-access-sa \
  --namespace default \
  --cluster ai-labs-cluster \
  --region us-east-1 \
  --attach-policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess \
  --approve
```

### Step 3 — Deploy a pod using the service account

```bash
cat > s3-test-pod.yaml << EOF
apiVersion: v1
kind: Pod
metadata:
  name: s3-test-pod
spec:
  restartPolicy: Never
  serviceAccountName: s3-access-sa
  containers:
  - name: aws-cli
    image: amazon/aws-cli
    command: ["aws", "s3", "ls"]
EOF

kubectl apply -f s3-test-pod.yaml
kubectl logs s3-test-pod
```

The pod lists your S3 buckets without any credentials — IAM roles for service accounts in action.

---

## 🔧 Part 5 — Monitoring with kubectl

Learn these commands — you'll use them daily:

```bash
# Describe a pod in detail (great for debugging)
kubectl describe pod <pod-name>

# View logs
kubectl logs <pod-name>

# Follow logs live
kubectl logs <pod-name> -f

# Get events (what's happening in the cluster)
kubectl get events --sort-by=.metadata.creationTimestamp

# Resource usage
kubectl top nodes
kubectl top pods
```

---

## 🧹 Cleanup — Important!

EKS costs money while running. Delete when done:

```bash
# Delete the service (removes load balancer)
kubectl delete service demo-app-service

# Delete the cluster
eksctl delete cluster --name ai-labs-cluster --region us-east-1
```

This takes 10-15 minutes. Verify in the AWS console that the cluster is gone.

---

## ✅ What you learned

- How to create and manage an EKS cluster
- How to write and apply Kubernetes manifests
- Self-healing, scaling, and autoscaling
- How to securely connect pods to S3 using IAM roles
- Key debugging commands for production use

---

## 💭 Reflection questions

1. What's the difference between a Deployment and a Pod? Why would you use one over the other?
2. How does EKS differ from running your own Kubernetes cluster on EC2?
3. An AI inference service is getting 10x normal traffic. Walk through how you'd handle that with what you've learned.
4. How would you deploy a new version of a model with zero downtime?

---

## 🐛 Common errors

| Error | Cause | Fix |
|-------|-------|-----|
| Cluster creation hangs past 25 minutes | CloudFormation error underneath | Check AWS Console → CloudFormation → ai-labs-cluster stack → Events tab for the real error |
| `kubectl` commands fail after cluster creation | kubeconfig not updated | Run `aws eks update-kubeconfig --name ai-labs-cluster --region us-east-1` |
| LoadBalancer EXTERNAL-IP stuck at `<pending>` | Normal — takes 2–3 minutes to provision | Keep watching with `--watch`; if it's still pending after 5 minutes check the service events |
| `s3-test-pod` goes into CrashLoopBackOff | Missing `restartPolicy: Never` — pod keeps restarting after exiting | Make sure your pod spec includes `restartPolicy: Never` |
| Nodes stuck in `NotReady` | Usually a VPC or IAM permissions issue | Check `kubectl describe node <node-name>` and the CloudFormation events |

---

## 🎯 What to put on your GitHub README

> "Provisioned a managed Kubernetes cluster on AWS EKS using eksctl, deployed containerized applications with horizontal pod autoscaling, and configured IAM roles for service accounts (IRSA) to give pods secure S3 access without credentials."

---

## 📝 My notes

> Add your own observations, what broke, what surprised you, and what you learned here.
