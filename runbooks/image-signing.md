# Runbook: Image Signing and Verification with Cosign & Sigstore Policy Controller

This runbook outlines how to sign container images with Cosign during the CI/CD pipeline and how to configure/troubleshoot image verification via the Admission Controller.

## 1. Overview
To ensure supply chain security, the cluster blocks any deployment of unsigned images matching the pattern `ghcr.io/dungoppai/*` and `ghcr.io/vuong-bach/*` in namespaces with the label `policy.sigstore.dev/include=true`.

- **Signing:** Handled in GitHub Actions using `sigstore/cosign-installer` and `cosign sign`.
- **Enforcement:** Enforced at admission time by Sigstore Policy Controller using a `ClusterImagePolicy` that points to the public key at `signing/cosign.pub`.

## 2. Setting Up GitHub Actions Secret

To enable automated signing in CI, you must add the Cosign private key to your repository's secrets:
1. Go to your GitHub repository -> **Settings** -> **Secrets and variables** -> **Actions**.
2. Click **New repository secret**.
3. Name: `COSIGN_PRIVATE_KEY`
4. Value: Copy the entire contents of your local `cosign.key` file (including `-----BEGIN ENCRYPTED SIGSTORE PRIVATE KEY-----` and `-----END ENCRYPTED SIGSTORE PRIVATE KEY-----`).
5. Click **Add secret**.
6. Create another secret named `COSIGN_PASSWORD` with the password used when generating the key pair (e.g. `adminpassword`).

## 3. Manual Signing (For Emergencies or Local Builds)

If you need to manually sign an image:
```bash
$env:COSIGN_PASSWORD="adminpassword"  # Or your chosen passphrase
.\cosign.exe sign --key cosign.key ghcr.io/yourusername/w10-api:0.0.1
```

## 4. Troubleshooting Image Rejections

If a deployment fails with an error similar to:
```
admission webhook "policy.sigstore.dev" denied the request: validation failed: image ... does not match any authority
```

### Step A: Verify if namespace is opting-in to Policy Controller
Only namespaces with the label `policy.sigstore.dev/include=true` are verified. Check namespace labels:
```bash
kubectl get ns demo --show-labels
```
If the label is missing, apply it:
```bash
kubectl label namespace demo policy.sigstore.dev/include=true
```

### Step B: Verify the Image Signature
You can verify the signature of any image using your public key:
```bash
.\cosign.exe verify --key signing/cosign.pub ghcr.io/dungoppai/w10-api:0.0.1
```
If this fails, the image was not signed, or signed with a different key. Check the GitHub Actions run logs to see why the signing step failed.
