# Karpenter AWS Demo (20 minutes to complete)

## Demo

### Apply YAML and Watch

``` 

wget https://raw.githubusercontent.com/ellistarn/karpenter-aws-demo/main/capacity-reservations/autoscaler.yaml

NODE_GROUP_ARN=$(aws eks describe-nodegroup --nodegroup-name demo --cluster-name $USER-karpenter-aws-demo --output json | jq -r ".nodegroup.nodegroupArn") \
envsubst < autoscaler.yaml | kubectl apply -f -

# Open in 5 separate terminals
watch 'kubectl get pods'
watch 'kubectl get nodes'
watch -d 'kubectl get metricsproducers.autoscaling.karpenter.sh demo -ojson | jq ".status.reservedCapacity"'
watch -d 'kubectl get horizontalautoscalers.autoscaling.karpenter.sh demo -ojson | jq ".status" | jq "del(.conditions)"'
watch -d 'kubectl get scalablenodegroups.autoscaling.karpenter.sh demo -ojson | jq "del(.status.conditions)"| jq ".spec, .status"'
```

### Scale the Pods and Nodes

``` 

wget https://raw.githubusercontent.com/ellistarn/karpenter-aws-demo/main/capacity-reservations/inflate.yaml

REPLICAS=2 envsubst < inflate.yaml | kubectl apply -f -
REPLICAS=10 envsubst < inflate.yaml | kubectl apply -f -
REPLICAS=30 envsubst < inflate.yaml | kubectl apply -f -
```

## Cleanup

``` 

eksctl delete cluster --name ${CLUSTER_NAME}
aws iam delete-policy --policy-arn arn:aws:iam::${AWS_ACCOUNT_ID}:policy/Karpenter
```
