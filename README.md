# Exercise 13 – Secret Rotation Outage

## 📌 Objective

Simulate a production incident where an application fails authentication after a secret rotation because the updated secret is not propagated to the running application.

---

# 🏗️ Architecture

```
AWS Secrets Manager (Simulated)
            │
            ▼
   Kubernetes Secret
            │
            ▼
   Payment Service Deployment
            │
            ▼
Authentication
```

---

# 🚨 Incident

After rotating the application secret:

```
401 Unauthorized
```

Application logs:

```
Token validation failed
```

Kubernetes Secret:

```
kubectl get secret payment-secret
```

Secret was still using the old token.

---

# Root Cause

The application was reading the Kubernetes Secret as an environment variable.

Although the Secret was updated, the running pod continued using the old environment variable until the Deployment was restarted.

---

# Prerequisites

- AWS Account
- EKS Cluster
- kubectl
- eksctl

---

# Project Structure

```
Secret Rotation Outage/
│
├── payment-secret.yaml
├── payment.yaml
├── README.md
└── Screenshots/
```

---

# Step 1 - Create Namespace

```bash
kubectl create namespace demo
```

---

# Step 2 - Create Kubernetes Secret

Apply the secret.

```bash
kubectl apply -f payment-secret.yaml
```

Verify

```bash
kubectl get secret payment-secret -n demo
```

---

# Step 3 - Deploy Payment Service

```bash
kubectl apply -f payment.yaml
```

Verify

```bash
kubectl get pods -n demo
```

---

# Step 4 - Observe Failure

Check logs.

```bash
kubectl logs -f deployment/payment-service -n demo
```

Output

```
401 Unauthorized

Token validation failed
```

---

# Step 5 - Verify Secret

Decode the secret.

```bash
kubectl get secret payment-secret \
-n demo \
-o jsonpath="{.data.API_TOKEN}" | base64 --decode
```

Output

```
old-token-123
```

---

# Step 6 - Rotate Secret

Update `payment-secret.yaml`

```yaml
apiVersion: v1
kind: Secret

metadata:
  name: payment-secret
  namespace: demo

type: Opaque

stringData:
  API_TOKEN: new-token-456
```

Apply

```bash
kubectl apply -f payment-secret.yaml
```

Verify

```bash
kubectl get secret payment-secret \
-n demo \
-o jsonpath="{.data.API_TOKEN}" | base64 --decode
```

Output

```
new-token-456
```

---

# Step 7 - Restart Deployment

```bash
kubectl rollout restart deployment payment-service -n demo
```

Wait for rollout.

```bash
kubectl rollout status deployment payment-service -n demo
```

---

# Step 8 - Verify Fix

Check logs.

```bash
kubectl logs -f deployment/payment-service -n demo
```

Output

```
Authentication Successful
```

---

# Investigation Commands

Check Secret

```bash
kubectl get secret payment-secret -n demo
```

Describe Secret

```bash
kubectl describe secret payment-secret -n demo
```

Decode Secret

```bash
kubectl get secret payment-secret \
-n demo \
-o jsonpath="{.data.API_TOKEN}" | base64 --decode
```

Restart Deployment

```bash
kubectl rollout restart deployment payment-service -n demo
```

Deployment Status

```bash
kubectl rollout status deployment/payment-service -n demo
```

Application Logs

```bash
kubectl logs -f deployment/payment-service -n demo
```

---

# Root Cause Analysis (RCA)

## Problem

Application authentication started failing after secret rotation.

## Investigation

- Verified application logs.
- Checked Kubernetes Secret.
- Decoded the stored token.
- Found the application using the old token.
- Updated the Kubernetes Secret.
- Restarted the Deployment.

## Root Cause

The application consumed the secret through environment variables.

Environment variables are loaded only during container startup.

Updating the Kubernetes Secret does **not** automatically update running containers.

A Deployment restart was required to reload the new secret.

---

# Resolution

- Updated Kubernetes Secret.
- Verified the rotated token.
- Restarted Deployment.
- Confirmed successful authentication.

---

# Skills Demonstrated

- Kubernetes Secrets
- Secret Rotation
- Base64 Secret Decoding
- Kubernetes Deployment
- Rollout Restart
- Authentication Troubleshooting
- Root Cause Analysis (RCA)
- Production Incident Investigation

---

# Key Learnings

- Updating a Kubernetes Secret does not automatically update running pods.
- Applications using environment variables require a restart after secret rotation.
- Always verify the secret before restarting the application.
- Root cause analysis helps identify synchronization issues during secret rotation.

---

# Cleanup

Delete Deployment

```bash
kubectl delete deployment payment-service -n demo
```

Delete Secret

```bash
kubectl delete secret payment-secret -n demo
```

Delete Namespace

```bash
kubectl delete namespace demo
```

Delete Cluster

```bash
eksctl delete cluster \
--name secret-rotation \
--region us-east-1
```

---

## Author

**Dharshan R**

B.Tech Information Technology

DevOps | AWS | Kubernetes | Docker | Jenkins | Terraform
