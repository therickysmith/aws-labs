# Lab 5 — SageMaker: AI/ML Model Deployment

**Time:** ~2 hours  
**Cost:** ~$0.05–0.10 — use ml.t2.medium, delete the endpoint when done  
**Goal:** Deploy a real ML model on AWS SageMaker. Understand the full lifecycle: train → package → deploy → call → monitor → clean up.

---

## 🧠 What is SageMaker?

SageMaker is AWS's managed ML platform. It handles training, hosting, and monitoring models at scale. As a DevOps/AI infrastructure engineer, you'll be building pipelines that feed into and out of SageMaker — or systems that replicate what it does. Understanding the SageMaker model is essential for understanding how AI companies think about inference infrastructure.

---

## 🏗️ Architecture

```
train_model.py
      │
      ▼
model.tar.gz ──► S3 bucket (models/)
                      │
                      ▼
              SageMaker endpoint
              (SKLearn container)
                      │
                      ▼
              test_endpoint.py
              (boto3 invoke_endpoint)
```

---

## 🛠️ Prerequisites

- Labs 1–4 complete
- Python 3.8+ and pip installed
- AWS CLI configured with a user that has SageMaker and S3 permissions

```bash
pip install boto3 sagemaker scikit-learn pandas numpy
```

---

## 🔧 Part 1 — Set up

### Step 1 — Create an S3 bucket

Pick a suffix to make your bucket name unique. Write it down — you'll use it throughout this lab and in Lab 6.

```bash
# Replace [suffix] with something like your initials + a number, e.g. rs42
aws s3 mb s3://sagemaker-labs-[suffix] --region us-east-1
```

### Step 2 — Create a SageMaker execution role

1. Go to IAM → Roles → Create role
2. Trusted entity: AWS service → SageMaker
3. Attach policy: **AmazonSageMakerFullAccess**
4. Name: `sagemaker-execution-role`
5. Create it, then click into the role and copy the **Role ARN** — you'll need it in Part 4

---

## 🔧 Part 2 — Train and upload a model

Create `train_model.py`:

```python
# train_model.py
import boto3
import numpy as np
from sklearn.ensemble import RandomForestClassifier
from sklearn.model_selection import train_test_split
import pickle
import os
import tarfile

BUCKET = "sagemaker-labs-[suffix]"  # update this

# Generate a simple classification dataset
np.random.seed(42)
X = np.random.randn(1000, 5)
y = (X[:, 0] + X[:, 1] > 0).astype(int)

X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2)
model = RandomForestClassifier(n_estimators=100)
model.fit(X_train, y_train)

accuracy = model.score(X_test, y_test)
print(f"Model accuracy: {accuracy:.3f}")

# Save model artifact
os.makedirs("model", exist_ok=True)
with open("model/model.pkl", "wb") as f:
    pickle.dump(model, f)

# SageMaker requires a .tar.gz archive
with tarfile.open("model.tar.gz", "w:gz") as tar:
    tar.add("model/model.pkl", arcname="model.pkl")

s3 = boto3.client("s3", region_name="us-east-1")

# Upload the SageMaker artifact
s3.upload_file("model.tar.gz", BUCKET, "models/model.tar.gz")
print(f"Uploaded model.tar.gz to s3://{BUCKET}/models/model.tar.gz")

# Also upload the raw pkl — Lab 6 pulls this directly
s3.upload_file("model/model.pkl", BUCKET, "models/model.pkl")
print(f"Uploaded model.pkl to s3://{BUCKET}/models/model.pkl")
```

Run it:
```bash
python train_model.py
```

Expected output:
```
Model accuracy: 0.xxx
Uploaded model.tar.gz to s3://sagemaker-labs-[suffix]/models/model.tar.gz
Uploaded model.pkl to s3://sagemaker-labs-[suffix]/models/model.pkl
```

---

## 🔧 Part 3 — Create the inference script

SageMaker wraps your model in a managed container and calls these four functions for every request. Create `inference.py` **in the same directory as `deploy_model.py`** — the SageMaker SDK packages it automatically at deploy time.

```python
# inference.py
import pickle
import numpy as np
import json
import os

def model_fn(model_dir):
    """Called once at startup — load and return the model."""
    with open(os.path.join(model_dir, "model.pkl"), "rb") as f:
        return pickle.load(f)

def input_fn(request_body, content_type):
    """Parse the raw request body into model input."""
    return np.array(json.loads(request_body))

def predict_fn(input_data, model):
    """Run inference."""
    return model.predict(input_data)

def output_fn(prediction, response_content_type):
    """Serialize the prediction back to the caller."""
    return json.dumps(prediction.tolist())
```

---

## 🔧 Part 4 — Deploy to a SageMaker endpoint

Create `deploy_model.py` in the same directory as `inference.py`:

```python
# deploy_model.py
import boto3
import sagemaker
from sagemaker.sklearn.model import SKLearnModel

ROLE_ARN = "arn:aws:iam::YOUR_ACCOUNT_ID:role/sagemaker-execution-role"  # update this
BUCKET = "sagemaker-labs-[suffix]"  # update this
REGION = "us-east-1"

boto_session = boto3.Session(region_name=REGION)
sagemaker_session = sagemaker.Session(boto_session=boto_session)

model = SKLearnModel(
    model_data=f"s3://{BUCKET}/models/model.tar.gz",
    role=ROLE_ARN,
    entry_point="inference.py",
    framework_version="1.0-1",
    sagemaker_session=sagemaker_session
)

print("Deploying — this takes 5–10 minutes...")
predictor = model.deploy(
    initial_instance_count=1,
    instance_type="ml.t2.medium"
)

print(f"\nEndpoint name: {predictor.endpoint_name}")
print("Save this — you'll need it for testing and cleanup.")
```

Run it:
```bash
python deploy_model.py
```

When it finishes, **copy the endpoint name** from the output. You'll paste it into the next two scripts.

---

## 🔧 Part 5 — Call the endpoint

Create `test_endpoint.py`:

```python
# test_endpoint.py
import boto3
import json
import numpy as np

ENDPOINT_NAME = "your-endpoint-name"  # paste from deploy output
REGION = "us-east-1"

client = boto3.client("sagemaker-runtime", region_name=REGION)

test_data = np.random.randn(5, 5).tolist()

response = client.invoke_endpoint(
    EndpointName=ENDPOINT_NAME,
    ContentType="application/json",
    Body=json.dumps(test_data)
)

result = json.loads(response["Body"].read().decode())
print(f"Predictions: {result}")
```

Run it:
```bash
python test_endpoint.py
```

You just called a live ML model endpoint over HTTPS. This is the same pattern AI companies use for LLM inference — the model is behind an endpoint, your code calls it.

---

## 🔧 Part 6 — Monitor the endpoint

```bash
# Check endpoint status
aws sagemaker describe-endpoint \
  --endpoint-name your-endpoint-name \
  --region us-east-1

# View invocation count in CloudWatch (last hour)
aws cloudwatch get-metric-statistics \
  --namespace AWS/SageMaker \
  --metric-name Invocations \
  --dimensions Name=EndpointName,Value=your-endpoint-name \
  --start-time $(date -u -d "1 hour ago" +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 3600 \
  --statistics Sum \
  --region us-east-1
```

---

## 🧹 Cleanup — Do this now

**Endpoints cost money every hour they run.** Delete it as soon as you're done.

```python
# cleanup.py
import boto3

ENDPOINT_NAME = "your-endpoint-name"  # paste from deploy output
REGION = "us-east-1"

sagemaker = boto3.client("sagemaker", region_name=REGION)
sagemaker.delete_endpoint(EndpointName=ENDPOINT_NAME)
print(f"Endpoint {ENDPOINT_NAME} deleted.")
```

```bash
python cleanup.py
```

Verify it's gone:
```bash
aws sagemaker list-endpoints --region us-east-1
```

---

## 🐛 Common errors

| Error | Cause | Fix |
|-------|-------|-----|
| `AccessDeniedException` on deploy | Execution role missing permissions | Make sure `AmazonSageMakerFullAccess` is attached to `sagemaker-execution-role` |
| `ResourceLimitExceeded` | Account's ml.t2.medium quota is 0 | Request a limit increase: Service Quotas → SageMaker → "ml.t2.medium for endpoint usage" |
| `EndpointAlreadyExists` | You ran deploy twice without cleaning up | Delete the old endpoint first, or change the endpoint name |
| `inference.py not found` | `inference.py` was in a different directory | Both files must be in the same folder when you run `deploy_model.py` |

---

## ✅ What you learned

- How to package and upload a trained model to S3 for SageMaker
- How the SageMaker inference lifecycle works: `model_fn → input_fn → predict_fn → output_fn`
- How to deploy and invoke a real endpoint using boto3
- How to monitor endpoint health with CloudWatch
- How to clean up to avoid unexpected charges

---

## 💭 Reflection questions

1. How is calling a SageMaker endpoint different from calling an API you built yourself on EC2 or EKS? What does SageMaker handle for you?
2. What would break if you forgot to implement `input_fn` and `output_fn` and why?
3. How would you deploy a new model version to this endpoint without any downtime?
4. SageMaker manages scaling for you. If you were running inference on EKS instead (as in Lab 6), what would you need to build yourself?

---

## 🎯 What to put on your GitHub README

> "Trained a scikit-learn classification model, packaged it as a SageMaker artifact, and deployed it to a live endpoint using the SageMaker Python SDK. Implemented the full SageMaker inference lifecycle (model_fn, input_fn, predict_fn, output_fn) and invoked the endpoint via boto3."

---

## 📝 My notes

> Add your own observations, what broke, what surprised you, and what you learned here.
