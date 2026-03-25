# WVA Autoscaling Troubleshooting Examples

## Example 1: Unhealthy Decode Deployment

**Scenario**: Decode pods are in CrashLoopBackOff, auto-detection fails.

**Solution**: Get model ID from prefill pods and create manually.

```bash
# Get model ID from running prefill deployment
MODEL_ID=$(kubectl exec -n dolev-llmd deployment/ms-qwen35b-llm-d-modelservice-prefill -- \
  curl -s localhost:8000/v1/models | jq -r '.data[0].id')

# Create VariantAutoscaling manually
bash skills/llmd-scale-workers/scripts/create-variantautoscaling-manual.sh \
  dolev-llmd \
  ms-qwen35b-autoscaler \
  ms-qwen35b-llm-d-modelservice-decode \
  "$MODEL_ID"
```

## Example 2: RBAC Permission Errors

**Scenario**: Controller logs show "pods is forbidden" errors.

**Solution**: Fix RBAC permissions.

```bash
# Fix RBAC
bash skills/llmd-scale-workers/scripts/fix-wva-rbac.sh dolev-llmd

# Verify no more errors
kubectl logs -n dolev-llmd deployment/workload-variant-autoscaler-controller-manager --tail=20
```

## Example 3: Complete Setup from Scratch

**Workflow**: Set up autoscaling for existing deployment.

```bash
NAMESPACE="dolev-llmd"

# 1. Detect deployment
bash skills/llmd-scale-workers/scripts/detect-deployment.sh $NAMESPACE

# 2. Check if WVA controller exists
kubectl get deployment -n $NAMESPACE workload-variant-autoscaler-controller-manager

# 3. Get model ID (from any running pod)
MODEL_ID=$(kubectl exec -n $NAMESPACE deployment/ms-qwen35b-llm-d-modelservice-prefill -- \
  curl -s localhost:8000/v1/models | jq -r '.data[0].id')

# 4. Create VariantAutoscaling
bash skills/llmd-scale-workers/scripts/create-variantautoscaling-manual.sh \
  $NAMESPACE \
  ms-qwen35b-autoscaler \
  ms-qwen35b-llm-d-modelservice-decode \
  "$MODEL_ID"

# 5. Fix RBAC if needed
bash skills/llmd-scale-workers/scripts/fix-wva-rbac.sh $NAMESPACE

# 6. Fix labels if needed
bash skills/llmd-scale-workers/scripts/fix-controller-instance-labels.sh $NAMESPACE

# 7. Monitor
kubectl get variantautoscaling -n $NAMESPACE -w
```

## Quick Reference

### Get Model ID
```bash
# From prefill (port 8000)
kubectl exec -n NAMESPACE deployment/PREFILL_NAME -- curl -s localhost:8000/v1/models | jq -r '.data[0].id'

# From decode (port 8200)
kubectl exec -n NAMESPACE deployment/DECODE_NAME -- curl -s localhost:8200/v1/models | jq -r '.data[0].id'
```

### Check Controller Status
```bash
kubectl get deployment -n NAMESPACE workload-variant-autoscaler-controller-manager
kubectl logs -n NAMESPACE deployment/workload-variant-autoscaler-controller-manager --tail=50
```

### Verify Autoscaling
```bash
kubectl get variantautoscaling -n NAMESPACE
kubectl describe variantautoscaling -n NAMESPACE AUTOSCALER_NAME