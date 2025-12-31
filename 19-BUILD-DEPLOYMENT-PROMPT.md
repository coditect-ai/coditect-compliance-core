---
title: 'Prompt 19: Component Build - Deployment Infrastructure'
type: reference
component_type: reference
version: 1.0.0
created: '2025-12-27'
updated: '2025-12-27'
status: active
tags:
- ai-ml
- authentication
- deployment
- security
- testing
- api
- architecture
- automation
summary: You are a senior DevOps engineer implementing the deployment infrastructure
  for CODITECT-COMPLIANCE. This includes Kubernetes manifests, Terraform...
moe_confidence: 0.950
moe_classified: 2025-12-31
---
# Prompt 19: Component Build - Deployment Infrastructure

## Context

You are a senior DevOps engineer implementing the deployment infrastructure for CODITECT-COMPLIANCE. This includes Kubernetes manifests, Terraform configurations, and CI/CD pipelines.

## Output Specification

Generate complete infrastructure-as-code for deploying CODITECT-COMPLIANCE. Output should be 1,500-2,500 lines of configuration.

## Implementation Requirements

### Technology Stack
- Google Kubernetes Engine (GKE)
- Terraform for infrastructure
- GitHub Actions for CI/CD
- Helm for Kubernetes packaging
- ArgoCD for GitOps

## Infrastructure Specifications

### 1. Terraform GCP Infrastructure

```hcl
# File: infrastructure/terraform/main.tf

terraform {
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
  }
  backend "gcs" {
    bucket = "coditect-terraform-state"
    prefix = "compliance"
  }
}

provider "google" {
  project = var.project_id
  region  = var.region
}

# GKE Cluster
resource "google_container_cluster" "compliance" {
  name     = "coditect-compliance-${var.environment}"
  location = var.region

  remove_default_node_pool = true
  initial_node_count       = 1

  workload_identity_config {
    workload_pool = "${var.project_id}.svc.id.goog"
  }

  private_cluster_config {
    enable_private_nodes    = true
    enable_private_endpoint = false
    master_ipv4_cidr_block  = "172.16.0.0/28"
  }
}

resource "google_container_node_pool" "primary" {
  name       = "primary-pool"
  cluster    = google_container_cluster.compliance.id
  node_count = var.node_count

  node_config {
    machine_type = var.machine_type
    disk_size_gb = 100
    disk_type    = "pd-ssd"

    oauth_scopes = [
      "https://www.googleapis.com/auth/cloud-platform"
    ]

    workload_metadata_config {
      mode = "GKE_METADATA"
    }
  }

  autoscaling {
    min_node_count = var.min_nodes
    max_node_count = var.max_nodes
  }
}

# Neo4j on GKE (via Helm)
resource "helm_release" "neo4j" {
  name       = "neo4j"
  repository = "https://helm.neo4j.com/neo4j"
  chart      = "neo4j"
  namespace  = "compliance"

  values = [
    file("${path.module}/helm-values/neo4j.yaml")
  ]
}

# Redis Cluster
resource "google_redis_instance" "compliance" {
  name           = "compliance-redis-${var.environment}"
  tier           = "STANDARD_HA"
  memory_size_gb = 5
  region         = var.region

  auth_enabled = true
  transit_encryption_mode = "SERVER_AUTHENTICATION"
}

# Cloud Storage for Evidence
resource "google_storage_bucket" "evidence" {
  for_each = toset(["us", "eu", "apac"])
  
  name     = "coditect-evidence-${each.key}-${var.environment}"
  location = local.bucket_locations[each.key]

  uniform_bucket_level_access = true
  
  encryption {
    default_kms_key_name = google_kms_crypto_key.evidence[each.key].id
  }

  lifecycle_rule {
    condition {
      age = 2555  # 7 years for audit
    }
    action {
      type = "Delete"
    }
  }
}

# Secret Manager for credentials
resource "google_secret_manager_secret" "integration_keys" {
  secret_id = "integration-encryption-key"
  
  replication {
    auto {}
  }
}
```

### 2. Kubernetes Manifests

```yaml
# File: infrastructure/k8s/base/deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: compliance-api
  namespace: compliance
spec:
  replicas: 3
  selector:
    matchLabels:
      app: compliance-api
  template:
    metadata:
      labels:
        app: compliance-api
    spec:
      serviceAccountName: compliance-api
      containers:
        - name: api
          image: gcr.io/coditect/compliance-api:latest
          ports:
            - containerPort: 8000
          env:
            - name: ENVIRONMENT
              valueFrom:
                configMapKeyRef:
                  name: compliance-config
                  key: environment
            - name: FDB_CLUSTER_FILE
              value: /etc/foundationdb/fdb.cluster
            - name: NEO4J_URI
              valueFrom:
                secretKeyRef:
                  name: neo4j-credentials
                  key: uri
            - name: REDIS_URL
              valueFrom:
                secretKeyRef:
                  name: redis-credentials
                  key: url
          resources:
            requests:
              cpu: "500m"
              memory: "1Gi"
            limits:
              cpu: "2"
              memory: "4Gi"
          livenessProbe:
            httpGet:
              path: /health
              port: 8000
            initialDelaySeconds: 10
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /ready
              port: 8000
            initialDelaySeconds: 5
            periodSeconds: 5
          volumeMounts:
            - name: fdb-config
              mountPath: /etc/foundationdb
      volumes:
        - name: fdb-config
          configMap:
            name: fdb-cluster-config

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: agent-worker
  namespace: compliance
spec:
  replicas: 5
  selector:
    matchLabels:
      app: agent-worker
  template:
    metadata:
      labels:
        app: agent-worker
    spec:
      containers:
        - name: worker
          image: gcr.io/coditect/compliance-worker:latest
          env:
            - name: WORKER_TYPE
              value: "agent"
            - name: ANTHROPIC_API_KEY
              valueFrom:
                secretKeyRef:
                  name: anthropic-credentials
                  key: api-key
          resources:
            requests:
              cpu: "1"
              memory: "2Gi"
            limits:
              cpu: "4"
              memory: "8Gi"

---
apiVersion: v1
kind: Service
metadata:
  name: compliance-api
  namespace: compliance
spec:
  selector:
    app: compliance-api
  ports:
    - port: 80
      targetPort: 8000
  type: ClusterIP

---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: compliance-ingress
  namespace: compliance
  annotations:
    kubernetes.io/ingress.class: "gce"
    kubernetes.io/ingress.global-static-ip-name: "compliance-ip"
    networking.gke.io/managed-certificates: "compliance-cert"
spec:
  rules:
    - host: api.compliance.coditect.ai
      http:
        paths:
          - path: /*
            pathType: ImplementationSpecific
            backend:
              service:
                name: compliance-api
                port:
                  number: 80
```

### 3. GitHub Actions CI/CD

```yaml
# File: .github/workflows/deploy.yaml

name: Deploy CODITECT-COMPLIANCE

on:
  push:
    branches: [main, staging]
  pull_request:
    branches: [main]

env:
  PROJECT_ID: coditect-production
  GKE_CLUSTER: coditect-compliance
  GKE_REGION: us-central1

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.12'
          
      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          pip install -r requirements-dev.txt
          
      - name: Run tests
        run: pytest tests/ --cov=src --cov-report=xml
        
      - name: Upload coverage
        uses: codecov/codecov-action@v4

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
        
      - name: Login to GCR
        uses: docker/login-action@v3
        with:
          registry: gcr.io
          username: _json_key
          password: ${{ secrets.GCP_SA_KEY }}
          
      - name: Build and push API
        uses: docker/build-push-action@v5
        with:
          context: .
          file: ./Dockerfile.api
          push: true
          tags: gcr.io/${{ env.PROJECT_ID }}/compliance-api:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy-staging:
    needs: build
    if: github.ref == 'refs/heads/staging'
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up gcloud
        uses: google-github-actions/setup-gcloud@v2
        with:
          service_account_key: ${{ secrets.GCP_SA_KEY }}
          
      - name: Get GKE credentials
        run: |
          gcloud container clusters get-credentials $GKE_CLUSTER \
            --region $GKE_REGION --project $PROJECT_ID
            
      - name: Deploy to staging
        run: |
          kubectl set image deployment/compliance-api \
            api=gcr.io/$PROJECT_ID/compliance-api:${{ github.sha }} \
            -n compliance-staging

  deploy-production:
    needs: build
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy to production
        run: |
          # ArgoCD sync for production
          argocd app sync compliance-production --grpc-web
```

### 4. Helm Chart

```yaml
# File: infrastructure/helm/compliance/values.yaml

replicaCount:
  api: 3
  worker: 5
  scheduler: 1

image:
  repository: gcr.io/coditect/compliance
  tag: latest
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: true
  className: gce
  hosts:
    - host: api.compliance.coditect.ai
      paths:
        - path: /
          pathType: Prefix

resources:
  api:
    requests:
      cpu: 500m
      memory: 1Gi
    limits:
      cpu: 2
      memory: 4Gi
  worker:
    requests:
      cpu: 1
      memory: 2Gi
    limits:
      cpu: 4
      memory: 8Gi

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 20
  targetCPUUtilizationPercentage: 70

neo4j:
  enabled: true
  core:
    numberOfServers: 3
  
redis:
  enabled: false  # Using managed Redis

monitoring:
  enabled: true
  prometheus:
    enabled: true
  grafana:
    enabled: true
```

## File Structure

```
infrastructure/
├── terraform/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   ├── gke.tf
│   ├── storage.tf
│   ├── networking.tf
│   └── environments/
│       ├── staging.tfvars
│       └── production.tfvars
├── k8s/
│   ├── base/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── configmap.yaml
│   └── overlays/
│       ├── staging/
│       └── production/
├── helm/
│   └── compliance/
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/
└── .github/
    └── workflows/
        ├── deploy.yaml
        └── terraform.yaml
```

## Acceptance Criteria

1. **Terraform**: Complete GCP infrastructure
2. **Kubernetes**: Production-ready manifests
3. **CI/CD**: Automated test and deploy
4. **Helm**: Configurable chart
5. **GitOps**: ArgoCD integration

## Token Budget

- Target: 15,000-20,000 tokens

## Dependencies

- Input: All component implementations
- Output: Deployed production system
