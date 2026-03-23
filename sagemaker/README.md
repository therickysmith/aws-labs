# Lab 5 — SageMaker: AI/ML Model Deployment

**Time:** ~2 hours  
**Cost:** ~$0.05-0.10 for the lab — use ml.t2.medium instances  
**Goal:** Deploy a real ML model on AWS SageMaker. Understand the full lifecycle from model to endpoint.

---

## 🧠 What is SageMaker?

SageMaker is AWS's managed ML platform. It handles the entire ML lifecycle — training, hosting, and monitoring models. AI companies use it (or build similar systems) to deploy models at scale. As a DevOps engineer, your job is to build the pipelines that feed into and out of SageMaker.

---

## 🛠️ Prerequisites

- Labs 1-4 complete
- Python 3.8+ installed
- pip installed

```bash
pip install boto3 sagemaker scikit-learn pandas numpy
```

---

## 🔧 Part 1 — Set up SageMaker

### Step 1 — Create an S3 bucket for SageMaker

```bash
aws s3 mb s3://ricky-sagemaker-labs-[random] --region us-east-1
```

### Step 2 — Create a SageMaker execution role
1. Go to IAM → Roles → Create role
2. Trusted entity: AWS service → SageMaker
3. Attach policy: **AmazonSageMakerFullAccess**
4. Name: `sagemaker-execution-role`
5. Copy the Role ARN — you'll need it

---

## 🔧 Part 2 — Train a simple model

Create a Python script that trains and uploads a model:

```python
# train_model.py
import boto3
import pandas as pd
import numpy as np
from sklearn.ensemble import RandomForestClassifier
from sklearn.model_selection import train_test_split
import pickle
import os
import tarfile

# Create sample dataset (simulating AI classification task)
np.random.seed(42)
X = np.random.randn(1000, 5)
y = (X[:, 0] + X[:, 1] > 0).astype(int)

# Train model
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2)
model = RandomForestClassifier(n_estimators=100)
model.fit(X_train, y_train)

accuracy = model.score(X_test, y_test)
print(f"Model accuracy: {accuracy:.3f}")

# Save model
os.makedirs("model", exist_ok=True)
with open("model/model.pkl", "wb") as f:
    pickle.dump(model, f)

# Package for SageMaker (must be model.tar.gz)
with tarfile.open("model.tar.gz", "w:gz") as tar:
    tar.add("model/model.pkl", arcname="model.pkl")

print("Model saved to model.tar.gz")

# Upload to S3
s3 = boto3.client("s3", region_name="us-east-1")
bucket = "ricky-sagemaker-labs-[your-random]"  # update this
s3.upload_file("model.tar.gz", bucket, "models/model.tar.gz")
print(f"Model uploaded to s3://{bucket}/models/model.tar.gz")
```

Run it:
```bash
python train_model.py
```

---

## 🔧 Part 3 — Deploy model to a SageMaker endpoint

```python
# deploy_model.py
import boto3
import sagemaker
from sagemaker.sklearn.model import SKLearnModel

# Config
ROLE_ARN = "arn:aws:iam::YOUR_ACCOUNT:role/sagemaker-execution-role"
BUCKET = "ricky-sagemaker-labs-[your-random]"
MODEL_URI = f"s3://{BUCKET}/models/model.tar.gz"
REGION = "us-east-1"

# Create SageMaker session
boto_session = boto3.Session(region_name=REGION)
sagemaker_session = sagemaker.Session(boto_session=boto_session)

# Create model
model = SKLearnModel(
    model_data=MODEL_URI,
    role=ROLE_ARN,
    entry_point="inference.py",
    framework_version="1.0-1",
    sagemaker_session=sagemaker_session
)

# Deploy endpoint
print("Deploying model... (takes 5-10 minutes)")
predictor = model.deploy(
    initial_instance_count=1,
    instance_type="ml.t2.medium"
)

print(f"Endpoint deployed: {predictor.endpoint_name}")
```

Create the inference script SageMaker needs:

```python
# inference.py
import pickle
import numpy as np
import os

def model_fn(model_dir):
    """Load model from the model_dir."""
    with open(os.path.join(model_dir, "model.pkl"), "rb") as f:
        model = pickle.load(f)
    return model

def predict_fn(input_data, model):
    """Make prediction."""
    return model.predict(input_data)

def input_fn(request_body, request_content_type):
    """Parse input data."""
    import json
    data = json.loads(request_body)
    return np.array(data)

def output_fn(prediction, response_content_type):
    """Format output."""
    import json
    return json.dumps(prediction.tolist())
```

Run deployment:
```bash
python deploy_model.py
```

---

## 🔧 Part 4 — Call your endpoint

```python
# test_endpoint.py
import boto3
import json
import numpy as np

ENDPOINT_NAME = "your-endpoint-name"  # from deploy output
REGION = "us-east-1"

# Create runtime client
client = boto3.client("sagemaker-runtime", region_name=REGION)

# Test data
test_data = np.random.randn(5, 5).tolist()

# Call endpoint
response = client.invoke_endpoint(
    EndpointName=ENDPOINT_NAME,
    ContentType="application/json",
    Body=json.dumps(test_data)
)

result = json.loads(response["Body"].read().decode())
print(f"Predictions: {result}")
```

You just called a real ML model endpoint on AWS. This is the same pattern used for LLM inference at AI companies.

---

## 🔧 Part 5 — Monitor your endpoint

```bash
# Check endpoint status
aws sagemaker describe-endpoint \
  --endpoint-name your-endpoint-name \
  --region us-east-1

# View CloudWatch metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/SageMaker \
  --metric-name Invocations \
  --dimensions Name=EndpointName,Value=your-endpoint-name \
  --start-time 2024-01-01T00:00:00Z \
  --end-time 2024-12-31T23:59:59Z \
  --period 3600 \
  --statistics Sum \
  --region us-east-1
```

---

## 🧹 Cleanup — Important!

Endpoints cost money while running:

```python
# cleanup.py
import boto3

sagemaker = boto3.client("sagemaker", region_name="us-east-1")

# Delete endpoint
sagemaker.delete_endpoint(EndpointName="your-endpoint-name")
print("Endpoint deleted")
```

---

## ✅ What you learned

- How to package and upload a model to S3
- How to deploy a model to a SageMaker endpoint
- How to call a model endpoint via API
- How monitoring works for ML endpoints

---

## 💭 Reflection questions

1. How is deploying a model endpoint similar to deploying a web service? How is it different?
2. What would you monitor in production to know if a model endpoint is healthy?
3. How would you handle a rolling update to deploy a new model version with zero downtime?
4. What's the difference between SageMaker and running your own inference server on EKS?

---

## 📝 My notes

> Add your own observations, what broke, what surprised you, and what you learned here.
