# Deploying JFrog Artifactory HA on a Private Kubernetes Cluster (Air-Gapped)

This guide explains how to deploy **JFrog Artifactory High Availability (HA)** in an on-premises Kubernetes cluster **without internet access**, using **Helm**, an **external PostgreSQL** database, and **external S3 storage in Azure**.

---

## 1. Prerequisites

Before starting, ensure you have:

- A **private Kubernetes cluster** (air-gapped, no internet access).
- A **VM with internet access** to download Helm charts and Docker images.
- A **private container registry** (e.g., Harbor) accessible from your Kubernetes cluster.
- Access credentials for:
  - **External PostgreSQL**
  - **Azure Blob Storage S3-compatible endpoint**

---

## 2. Steps Overview

1. Download Helm chart and Docker images on an internet-enabled VM.
2. Push Docker images to your private registry.
3. Push Helm chart to your private Harbor Helm repository.
4. Configure `values.yaml` for external PostgreSQL and Azure S3.
5. Deploy Artifactory HA in your air-gapped cluster.

---

## 3. Download Helm Chart & Images

On the VM with internet:

```bash
# Add JFrog Helm repo
helm repo add jfrog https://charts.jfrog.io
helm repo update

# Download chart locally
helm pull jfrog/artifactory-ha --version 107.85.13 --untar
```

Extract all image references:

```bash
grep -R "image:" artifactory-ha/values.yaml | awk '{print $2}' | sort -u
```

Download Docker images:

```bash
while read img; do
  docker pull $img
  docker tag $img my-harbor.local/library/$(basename $img)
  docker push my-harbor.local/library/$(basename $img)
done < images.txt
```

---

## 4. Push Helm Chart to Private Harbor

```bash
helm package artifactory-ha
helm push artifactory-ha-107.85.13.tgz oci://my-harbor.local/helm
```

---

## 5. Configure values.yaml

Edit `values.yaml`:

```yaml
artifactory:
  database:
    type: postgresql
    driver: org.postgresql.Driver
    url: jdbc:postgresql://<POSTGRES_HOST>:5432/artifactory
    user: artifactory
    password: <POSTGRES_PASSWORD>

  persistence:
    type: objectStorage
    objectStorage:
      provider: aws-s3
      endpoint: https://<AZURE_BLOB_S3_ENDPOINT>
      region: <REGION>
      bucketName: <BUCKET_NAME>
      accessKey: <ACCESS_KEY>
      secretKey: <SECRET_KEY>

postgresql:
  enabled: false

nginx:
  enabled: true

imageRegistry: my-harbor.local/library
```

---

## 6. Deploy in Air-Gapped Cluster

```bash
helm repo add my-harbor oci://my-harbor.local/helm
helm install artifactory-ha my-harbor/artifactory-ha -f values.yaml
```

---

## 7. Verify Deployment

```bash
kubectl get pods -n artifactory
kubectl logs <pod_name> -n artifactory
```

---

## 8. Notes

- You must **synchronize all required Docker images** before deploying.
- The PostgreSQL and Azure S3 credentials must be correct before deployment.
- For production, consider enabling **TLS** and **ingress controller**.
