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

NODE_GROUP_ARN=$(aws eks describe-nodegroup --nodegroup-name demo --cluster-name ${CLUSTER_NAME} --output json | jq -r ".nodegroup.nodegroupArn") \
QUEUE_ARN=arn:aws:sqs:$REGION:$AWS_ACCOUNT_ID:$QUEUE_NAME \
envsubst < manifest.yaml | kubectl apply -f -

envsubst '$QUEUE_URL, $QUEUE_NAME' < queue-processor.yaml | kubectl apply -f -  

# Open in 4 separate terminals
watch -d "kubectl get pods -n karpenter | grep queue-processor | wc -l"
watch -d "kubectl get nodes"
watch -d 'kubectl get metricsproducers -ojson | jq ".items[0].status.queue"'
watch -d 'kubectl get scalablenodegroups -ojson | jq "del(.status.conditions)"| jq ".items[0].spec, .items[0].status"'
```

### Scale the Queue Up

```bash

# Send 10 messages to the queue
aws sqs send-message-batch --queue-url $QUEUE_URL --entries file://message-batch.json

# Send # of Replicas to the Queue. Send-message-batch can only send 10 messages. Use this command to send a large batch.
REPLICAS=30 envsubst "'$REPLICAS, $QUEUE_URL'" < messages.yaml | kubectl apply -f -
```
