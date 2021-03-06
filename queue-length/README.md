# Queue Length Demo

This demo will use the length of an AWS SQS Queue to autoscale a Deployment and Node Group.

We will deploy pods that handle 1 message every 30 seconds. The autoscalers are configured to create 4 pods per node.

## Environment

```bash
QUEUE_NAME=$USER-demo-queue 
QUEUE_URL=$(aws sqs create-queue --queue-name $QUEUE_NAME --output json | jq -r '.QueueUrl') 
```

## Apply YAML

```bash
wget https://raw.githubusercontent.com/ellistarn/karpenter-aws-demo/main/queue-length/manifest.yaml

QUEUE_NAME=$QUEUE_NAME \
QUEUE_URL=$QUEUE_URL \
QUEUE_ARN=arn:aws:sqs:$REGION:$AWS_ACCOUNT_ID:$QUEUE_NAME \
NODE_GROUP_ARN=$(aws eks describe-nodegroup --nodegroup-name karpenter-aws-demo --cluster-name ${CLUSTER_NAME} --output json | jq -r ".nodegroup.nodegroupArn") \
envsubst < manifest.yaml | kubectl apply -f -
```

## Grant SQSQueue permissions to the namespace's default service account

```bash
eksctl create iamserviceaccount --cluster ${CLUSTER_NAME} \
--name default \
--namespace karpenter-queue-length-demo \
--attach-policy-arn "arn:aws:iam::${AWS_ACCOUNT_ID}:policy/Karpenter" \
--override-existing-serviceaccounts \
--approve

kubectl delete pods -n karpenter -l control-plane=karpenter
kubectl get pods -n karpenter
```

## Watch demo

```bash
# Manually run these in 7 separate terminal windows
watch 'kubectl get pods -l app=subscriber -n karpenter-queue-length-demo'
watch 'kubectl get nodes -n karpenter-queue-length-demo'
watch -d 'kubectl get metricsproducer demo -n karpenter-queue-length-demo -ojson | jq .status.queue'
watch -d 'kubectl get horizontalautoscalers.autoscaling.karpenter.sh capacity -n karpenter-queue-length-demo -ojson | jq .status | jq del\(.conditions\)'
watch -d 'kubectl get horizontalautoscalers.autoscaling.karpenter.sh subscriber -n karpenter-queue-length-demo -ojson | jq .status | jq del\(.conditions\)'
watch -d 'kubectl get scalablenodegroup capacity -n karpenter-queue-length-demo -ojson | jq del\(.status.conditions\) | jq .spec,.status'
watch "kubectl get metricsproducers capacity-watcher -n karpenter-queue-length-demo -ojson | jq -r '.status.reservedCapacity'"
```

## Send messages to the queue

```bash
# Create 10 messages to send to SQS
read -r -d '' QUEUE_ENTRIES <<EOM
[
  {"Id": "0","MessageBody": "body"},{"Id": "1","MessageBody": "body"},{"Id": "2","MessageBody": "body"},{"Id": "3","MessageBody": "body"},
  {"Id": "4","MessageBody": "body"},{"Id": "5","MessageBody": "body"},{"Id": "6","MessageBody": "body"},{"Id": "7","MessageBody": "body"},
  {"Id": "8","MessageBody": "body"},{"Id": "9","MessageBody": "body"}
]
EOM

# Send messages to the queue every 10 seconds
while true; do
  aws sqs send-message-batch --queue-url $QUEUE_URL --entries "$QUEUE_ENTRIES"
  sleep 10
done
```

# Clean up

```bash
rm manifest.yaml

kubectl delete namespace karpenter-queue-length-demo
aws sqs delete-queue --queue-url $QUEUE_URL
# Clean up stacks
eksctl delete iamserviceaccount --cluster $CLUSTER_NAME --name default --namespace karpenter-queue-length-demo
```
