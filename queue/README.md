# Scaling with a Queue (10 minutes to complete)

This demo will use the length of an AWS SQS Queue to autoscale a Deployment and Node Group.

This demo scales 1 pod per message in the queue (inflate.yaml). It also scales up 1 node per 4 messages (base.yaml) in the queue.

## Setup

```bash

QUEUE_NAME=$USER-demo-queue

# If you don't already have a queue
QUEUE_URL=$(aws sqs create-queue --queue-name $QUEUE_NAME --output json | jq -r '.QueueUrl')

```

### Apply Scaling Infrastructure

```bash

wget https://raw.githubusercontent.com/ellistarn/karpenter-aws-demo/main/queue/manifest.yaml

NODE_GROUP_ARN=$(aws eks describe-nodegroup --nodegroup-name demo --cluster-name ${CLUSTER_NAME} --output json | jq -r ".nodegroup.nodegroupArn") \
QUEUE_ARN=arn:aws:sqs:$REGION:$AWS_ACCOUNT_ID:$QUEUE_NAME \
envsubst < manifest.yaml | kubectl apply -f -


wget https://raw.githubusercontent.com/ellistarn/karpenter-aws-demo/main/queue/queue-processor.yaml

envsubst '$QUEUE_URL, $QUEUE_NAME' < queue-processor.yaml | kubectl apply -f -  

# Open in 4 separate terminals
watch -d "kubectl get pods -n karpenter | grep queue-processor | wc -l"
watch -d "kubectl get nodes"
watch -d 'kubectl get metricsproducers -ojson | jq ".items[0].status.queue"'
watch -d 'kubectl get scalablenodegroups -ojson | jq "del(.status.conditions)"| jq ".items[0].spec, .items[0].status"'
```

### Scale the Queue Up

```bash

# Send 10 messages to the queue every 10 seconds
while true ;
do
aws sqs send-message-batch --queue-url $QUEUE_URL --entries "$(cat << EOM
[
  {"Id": "0","MessageBody": " "},{"Id": "1","MessageBody": " "},{"Id": "2","MessageBody": " "},{"Id": "3","MessageBody": " "},
  {"Id": "4","MessageBody": " "},{"Id": "5","MessageBody": " "},{"Id": "6","MessageBody": " "},{"Id": "7","MessageBody": " "},
  {"Id": "8","MessageBody": " "},{"Id": "9","MessageBody": " "}
]
EOM
)" ;
sleep 10; 
done
```
