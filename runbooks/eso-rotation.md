# Runbook: Secret Rotation using External Secrets Operator (ESO)

This runbook describes how to manage and rotate database secrets using ESO and verify that changes propagate dynamically to the application without causing a pod restart.

## 1. Overview
The API application consumes the database password dynamically from a volume mount (`/app/secrets/password`) rather than environment variables. 
The External Secrets Operator (ESO) automatically pulls secrets from the Secrets Manager (AWS/Fake provider) and keeps the Kubernetes Secret (`db-secret`) updated.

- **Sync Interval:** 10 seconds.
- **Propagation to Pod:** Kubelet updates the mounted volume automatically (typically takes 10–45 seconds).
- **Application Behavior:** The Flask app reads the password file on every request, ensuring instant pickup of the rotated secret without any pod restart.

## 2. How to Rotate the Secret (Fake Store Version)

Since we are utilizing the `Fake` provider for verification, you rotate the secret by changing the value defined in [secret-store.yaml](file:///d:/uni/xbrain/phase_2/w10/thur-lab/eso/secret-store.yaml):

1. Open `eso/secret-store.yaml` and update the value:
   ```yaml
   spec:
     provider:
       fake:
         data:
           - key: db-password
             value: "new-secret-password-xyz"
   ```
2. Commit and push the change to Git.
3. ArgoCD will sync the change to the cluster.

## 3. How to Rotate the Secret (AWS Secrets Manager Version)

In a live production environment using AWS:
1. Update the secret value via AWS Console or AWS CLI:
   ```bash
   aws secretsmanager update-secret --secret-id prod/db-password --secret-string "new-aws-password-value"
   ```
2. The `ExternalSecret` will fetch the new value on the next sync interval (10s) and update the local K8s Secret.

## 4. Verification Steps

To verify that the rotation successfully occurred within 60 seconds without restarts:

### A. Check the Kubernetes Secret
Verify that the secret in Kubernetes got updated with the new value:
```bash
kubectl get secret db-secret -n demo -o jsonpath="{.data.password}" | base64 --decode
```

### B. Verify Pod is NOT Restarted
Verify that the pod's AGE and restart count remain unchanged:
```bash
kubectl get pods -n demo -l app=api
```
*(Look at the `AGE` and `RESTARTS` columns. The pod age should represent when it was first deployed, not when the secret was updated.)*

### C. Verify the API returns the New Value
Port-forward to the API service:
```bash
kubectl port-forward svc/api -n demo 8080:8080
```
Query the `/` endpoint:
```bash
curl http://localhost:8080/
```
Verify that the JSON response contains the updated password:
```json
{
  "ok": true,
  "password": "new-secret-password-xyz",
  "version": "v0.0.1"
}
```
