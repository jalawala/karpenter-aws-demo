# Reserved Capacity Demo

## Apply YAML and Watch

```bash
wget https://raw.githubusercontent.com/ellistarn/karpenter-aws-demo/main/capacity-reservations/manifest.yaml

NODE_GROUP_ARN=$(aws eks describe-nodegroup --nodegroup-name demo --cluster-name $USER-karpenter-aws-demo --output json | jq -r ".nodegroup.nodegroupArn") \
envsubst < manifest.yaml | kubectl apply -f -

# Open in 5 separate terminals
watch 'kubectl get pods'
watch 'kubectl get nodes'
watch -d 'kubectl get metricsproducers.autoscaling.karpenter.sh demo -n karpenter-reserved-capacity-demo -ojson | jq ".status.reservedCapacity"'
watch -d 'kubectl get horizontalautoscalers.autoscaling.karpenter.sh capacity -n karpenter-reserved-capacity-demo -ojson | jq ".status" | jq "del(.conditions)"'
watch -d 'kubectl get scalablenodegroups.autoscaling.karpenter.sh capacity -n karpenter-reserved-capacity-demo -ojson | jq "del(.status.conditions)"| jq ".spec, .status"'
```

## Scale the Pods and Nodes

```bash
wget https://raw.githubusercontent.com/ellistarn/karpenter-aws-demo/main/capacity-reservations/inflate.yaml

REPLICAS=2 envsubst < inflate.yaml | kubectl apply -f -
REPLICAS=10 envsubst < inflate.yaml | kubectl apply -f -
REPLICAS=30 envsubst < inflate.yaml | kubectl apply -f -
```

## Cleanup
kubectl delete namespace karpenter-reserved-capacity-demo
