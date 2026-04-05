# Multi-Cluster GKE Inference Gateway

The GKE Inference Gateway is built on the Kubernetes Gateway API with inference-specific routing. With multi-cluster support, you deploy inference backends in multiple clusters and a single gateway distributes traffic across them with health-based failover.

This lab uses a mock inference server to keep costs low. In production you'd use vLLM or TGI.

## What this covers

- Registering two GKE clusters in a Fleet
- Deploying an inference backend on both clusters
- Configuring InferencePool and InferenceGateway
- Testing multi-cluster routing and failover

## Prerequisites

- `gcloud` CLI authenticated
- `kubectl` installed
- GCP project with billing enabled
- APIs enabled:

```bash
gcloud services enable \
  container.googleapis.com \
  gkehub.googleapis.com \
  multiclusterservicediscovery.googleapis.com \
  multiclusteringress.googleapis.com
```

## Set variables

```bash
export PROJECT_ID=$(gcloud config get-value project)
export CLUSTER_A=lab-inference-cluster-a
export CLUSTER_B=lab-inference-cluster-b
export REGION_A=us-central1
export REGION_B=europe-west1
```

## Clone the repo

```bash
git clone https://github.com/misskecupbung/gke-multi-cluster-inference-gateway.git
cd gke-multi-cluster-inference-gateway
```

## Step 1 — Create two clusters

```bash
gcloud container clusters create $CLUSTER_A \
  --region $REGION_A \
  --num-nodes 1 \
  --machine-type e2-standard-2 \
  --disk-size=50 \
  --disk-type=pd-standard \
  --release-channel rapid \
  --workload-pool ${PROJECT_ID}.svc.id.goog \
  --enable-inference-gateway

gcloud container clusters create $CLUSTER_B \
  --region $REGION_B \
  --num-nodes 1 \
  --machine-type e2-standard-2 \
  --disk-size=50 \
  --disk-type=pd-standard \
  --release-channel rapid \
  --workload-pool ${PROJECT_ID}.svc.id.goog \
  --enable-inference-gateway
```

---

## Step 2 — Register in a Fleet

```bash
gcloud container fleet memberships register $CLUSTER_A \
  --gke-cluster ${REGION_A}/${CLUSTER_A} \
  --enable-workload-identity

gcloud container fleet memberships register $CLUSTER_B \
  --gke-cluster ${REGION_B}/${CLUSTER_B} \
  --enable-workload-identity
```

```bash
gcloud container fleet memberships list
gcloud container fleet multi-cluster-services enable
```

## Step 3 — Set up kubectl contexts

```bash
gcloud container clusters get-credentials $CLUSTER_A --region $REGION_A
kubectl config rename-context gke_${PROJECT_ID}_${REGION_A}_${CLUSTER_A} cluster-a

gcloud container clusters get-credentials $CLUSTER_B --region $REGION_B
kubectl config rename-context gke_${PROJECT_ID}_${REGION_B}_${CLUSTER_B} cluster-b
```

```bash
kubectl --context cluster-a get nodes
kubectl --context cluster-b get nodes
```

## Step 4 — Deploy inference backend on both clusters

```bash
kubectl --context cluster-a apply -f manifests/inference-backend.yaml
kubectl --context cluster-a apply -f manifests/inference-service.yaml

kubectl --context cluster-b apply -f manifests/inference-backend.yaml
kubectl --context cluster-b apply -f manifests/inference-service.yaml
```

Wait for pods:

```bash
kubectl --context cluster-a get pods -w
kubectl --context cluster-b get pods -w
```

## Step 5 — Create InferencePools

```bash
kubectl --context cluster-a apply -f manifests/inference-pool.yaml
kubectl --context cluster-b apply -f manifests/inference-pool.yaml
```

## Step 6 — Create the gateway and route

```bash
kubectl --context cluster-a apply -f manifests/inference-gateway.yaml
kubectl --context cluster-a apply -f manifests/httproute.yaml
kubectl --context cluster-a get gateway -w
```

External IP appears in 2–5 minutes.

```bash
GATEWAY_IP=$(kubectl --context cluster-a get gateway inference-gateway \
  -o jsonpath='{.status.addresses[0].value}')
echo $GATEWAY_IP
```

## Step 7 — Test routing

```bash
for i in {1..10}; do
  curl -s -X POST http://$GATEWAY_IP/v1/completions \
    -H "Content-Type: application/json" \
    -d '{"model": "mock-model", "prompt": "Hello", "max_tokens": 10}' | \
    python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('served_by','unknown'))"
done
```

Traffic goes to both clusters.

## Step 8 — Test failover

```bash
kubectl --context cluster-a scale deployment inference-backend --replicas=0
```

```bash
for i in {1..5}; do
  curl -s -X POST http://$GATEWAY_IP/v1/completions \
    -H "Content-Type: application/json" \
    -d '{"model": "mock-model", "prompt": "Hello", "max_tokens": 10}' | \
    python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('served_by','unknown'))"
done
```

All traffic goes to cluster-b. Restore cluster-a:

```bash
kubectl --context cluster-a scale deployment inference-backend --replicas=2
```

## Step 9 — Clean up

```bash
kubectl --context cluster-a delete -f manifests/
kubectl --context cluster-b delete -f manifests/

gcloud container fleet memberships delete $CLUSTER_A --quiet
gcloud container fleet memberships delete $CLUSTER_B --quiet

gcloud container clusters delete $CLUSTER_A --region $REGION_A --quiet
gcloud container clusters delete $CLUSTER_B --region $REGION_B --quiet
```

## Limitations

- Doesn't solve GPU quota automatically. If both clusters are out of capacity, requests fail.
- No automatic geo-routing by client location. Current implementation is weighted routing with health-based failover.

## Resources

- [GKE Inference Gateway documentation](https://cloud.google.com/kubernetes-engine/docs/concepts/inference-gateway)
- [KV cache affinity routing docs](https://cloud.google.com/kubernetes-engine/docs/concepts/inference-gateway)
