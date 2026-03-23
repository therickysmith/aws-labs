# Lab 6 — AI Pipeline: End-to-End AI Deployment

**Time:** ~3-4 hours  
**Cost:** ~$0.50-1.00 total — clean up immediately after  
**Goal:** Build a complete end-to-end AI deployment pipeline combining everything from Labs 1-5. This is your portfolio centrepiece.

---

## 🧠 What you're building

A complete pipeline that:
1. Stores a model in S3
2. Deploys it to EKS as an inference service
3. Calls the Anthropic or OpenAI API to enrich responses
4. Exposes it via a load balancer
5. Has automated deployment via a shell script (simulating CI/CD)

This is a real, working AI deployment system. Put this on your GitHub and talk about it in every interview.

---

## 🏗️ Architecture

```
User Request
     │
     ▼
AWS Load Balancer
     │
     ▼
EKS Cluster
     │
     ├── Inference Pod (your model)
     │         │
     │         └── S3 (model weights)
     │
     └── API Gateway Pod
               │
               └── Anthropic/OpenAI API
```

---

## 🔧 Part 1 — Build the inference service

Create a Flask API that serves predictions:

```python
# app.py
from flask import Flask, request, jsonify
import pickle
import numpy as np
import os
import boto3
import anthropic

app = Flask(__name__)
model = None

def load_model():
    """Load model from S3 on startup."""
    global model
    bucket = os.environ.get("MODEL_BUCKET")
    key = os.environ.get("MODEL_KEY", "models/model.pkl")

    s3 = boto3.client("s3")
    s3.download_file(bucket, key, "/tmp/model.pkl")

    with open("/tmp/model.pkl", "rb") as f:
        model = pickle.load(f)
    print("Model loaded successfully")

@app.route("/health", methods=["GET"])
def health():
    return jsonify({"status": "healthy", "model_loaded": model is not None})

@app.route("/predict", methods=["POST"])
def predict():
    if model is None:
        return jsonify({"error": "Model not loaded"}), 503

    data = request.json.get("data")
    if not data:
        return jsonify({"error": "No data provided"}), 400

    # Get model prediction
    features = np.array(data)
    prediction = model.predict(features).tolist()

    # Enrich with AI explanation using Anthropic API
    client = anthropic.Anthropic(api_key=os.environ.get("ANTHROPIC_API_KEY"))
    message = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=150,
        messages=[{
            "role": "user",
            "content": f"A classification model predicted {prediction} for input features {data[0]}. Provide a one sentence plain-English explanation of what this prediction means."
        }]
    )
    explanation = message.content[0].text

    return jsonify({
        "prediction": prediction,
        "explanation": explanation,
        "model": "random-forest-v1"
    })

if __name__ == "__main__":
    load_model()
    app.run(host="0.0.0.0", port=5000)
```

---

## 🔧 Part 2 — Containerise the service

```dockerfile
# Dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

EXPOSE 5000

CMD ["python", "app.py"]
```

```text
# requirements.txt
flask==3.0.0
boto3==1.34.0
anthropic==0.20.0
scikit-learn==1.3.0
numpy==1.26.0
gunicorn==21.2.0
```

Build and push to AWS ECR:

```bash
# Create ECR repository
aws ecr create-repository \
  --repository-name ai-inference-service \
  --region us-east-1

# Get login token
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS \
  --password-stdin YOUR_ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com

# Build and push
docker build -t ai-inference-service .
docker tag ai-inference-service:latest \
  YOUR_ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/ai-inference-service:latest
docker push \
  YOUR_ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/ai-inference-service:latest
```

---

## 🔧 Part 3 — Deploy to EKS

```yaml
# kubernetes/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ai-inference
  labels:
    app: ai-inference
spec:
  replicas: 2
  selector:
    matchLabels:
      app: ai-inference
  template:
    metadata:
      labels:
        app: ai-inference
    spec:
      serviceAccountName: s3-access-sa
      containers:
      - name: ai-inference
        image: YOUR_ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/ai-inference-service:latest
        ports:
        - containerPort: 5000
        env:
        - name: MODEL_BUCKET
          value: "ricky-sagemaker-labs-[your-random]"
        - name: MODEL_KEY
          value: "models/model.pkl"
        - name: ANTHROPIC_API_KEY
          valueFrom:
            secretKeyRef:
              name: api-secrets
              key: anthropic-api-key
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 5000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: 5000
          initialDelaySeconds: 15
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: ai-inference-service
spec:
  selector:
    app: ai-inference
  ports:
    - protocol: TCP
      port: 80
      targetPort: 5000
  type: LoadBalancer
```

Create the API key secret:
```bash
kubectl create secret generic api-secrets \
  --from-literal=anthropic-api-key=YOUR_ANTHROPIC_API_KEY
```

Deploy:
```bash
kubectl apply -f kubernetes/deployment.yaml
kubectl get pods --watch
kubectl get service ai-inference-service
```

---

## 🔧 Part 4 — Automated deployment script

```bash
#!/bin/bash
# deploy.sh — simulates a CI/CD pipeline deployment

set -e

ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
REGION="us-east-1"
ECR_REPO="$ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com/ai-inference-service"
VERSION=$(date +%Y%m%d%H%M%S)

echo "🚀 Starting deployment v$VERSION"

# Step 1: Build
echo "📦 Building Docker image..."
docker build -t ai-inference-service:$VERSION .

# Step 2: Push to ECR
echo "⬆️  Pushing to ECR..."
aws ecr get-login-password --region $REGION | \
  docker login --username AWS --password-stdin $ECR_REPO
docker tag ai-inference-service:$VERSION $ECR_REPO:$VERSION
docker tag ai-inference-service:$VERSION $ECR_REPO:latest
docker push $ECR_REPO:$VERSION
docker push $ECR_REPO:latest

# Step 3: Rolling deploy to EKS
echo "☸️  Deploying to EKS..."
kubectl set image deployment/ai-inference \
  ai-inference=$ECR_REPO:$VERSION

# Step 4: Wait for rollout
echo "⏳ Waiting for rollout..."
kubectl rollout status deployment/ai-inference

# Step 5: Health check
echo "🔍 Running health check..."
ENDPOINT=$(kubectl get service ai-inference-service \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
sleep 10
curl -f http://$ENDPOINT/health

echo "✅ Deployment v$VERSION complete!"
echo "🌐 Endpoint: http://$ENDPOINT"
```

Make it executable and run:
```bash
chmod +x deploy.sh
./deploy.sh
```

---

## 🔧 Part 5 — Test the full pipeline

```bash
# Get your endpoint
ENDPOINT=$(kubectl get service ai-inference-service \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

# Health check
curl http://$ENDPOINT/health

# Make a prediction
curl -X POST http://$ENDPOINT/predict \
  -H "Content-Type: application/json" \
  -d '{"data": [[1.2, -0.5, 0.8, 1.1, -0.3]]}'
```

You should get back a prediction AND an AI-generated explanation. That's your full pipeline working end to end.

---

## 🧹 Cleanup

```bash
# Delete Kubernetes resources
kubectl delete -f kubernetes/deployment.yaml
kubectl delete secret api-secrets

# Delete EKS cluster
eksctl delete cluster --name ai-labs-cluster --region us-east-1

# Delete ECR repository
aws ecr delete-repository \
  --repository-name ai-inference-service \
  --force \
  --region us-east-1

# Empty and delete S3 bucket
aws s3 rm s3://your-bucket-name/ --recursive
aws s3api delete-bucket --bucket your-bucket-name
```

---

## ✅ What you built

A production-grade AI deployment pipeline that:
- Containerises an ML model as a REST API
- Deploys it to Kubernetes on AWS EKS
- Uses IAM roles for secure S3 access
- Integrates with the Anthropic API for AI enrichment
- Has automated deployment via a deploy script
- Has health checks and resource limits
- Uses Kubernetes secrets for sensitive config

---

## 💭 Reflection questions

1. How would you add autoscaling to handle traffic spikes to the inference endpoint?
2. What would you add to make this production-ready? (logging, tracing, alerting)
3. How would you handle model versioning — deploying v2 while v1 is still serving traffic?
4. What are the security improvements you'd make before putting this in front of real users?
5. How would you estimate the cost of running this at 1M requests/day?

---

## 🎯 What to put on your GitHub README

> "Built an end-to-end AI inference pipeline on AWS: containerised ML model → ECR → EKS deployment with autoscaling → integrated Anthropic API for response enrichment → automated rolling deployments via shell scripts. Uses IAM roles for secure S3 model storage, Kubernetes secrets for API key management, and liveness/readiness probes for production reliability."

That paragraph on your GitHub README will get attention from hiring managers at AI companies.

---

## 📝 My notes

> Add your own observations, what broke, what surprised you, and what you learned here.
