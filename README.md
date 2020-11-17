# Karpenter AWS Demo (20 minutes to complete)

## Setup

### Environment
```
CLOUD_PROVIDER=aws
AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
CLUSTER_NAME=$USER-karpenter-aws-demo
REGION=us-west-2
```

### Cluster
```
eksctl create cluster \
--name ${CLUSTER_NAME} \
--version 1.16 \
--region ${REGION} \
--nodegroup-name demo \
--node-type m5.2xlarge \
--nodes 1 \
--nodes-min 1 \
--nodes-max 10 \
--managed
```

### Karpenter Controller
```
TMP=$(mktemp -d)
trap "rm -rf $TMP" EXIT
(
    cd $TMP
    git clone https://github.com/awslabs/karpenter.git
    cd karpenter
    make toolchain
    make generate
    ./hack/quick-install.sh
)
rm -rf $TMP
```

### AWS Credentials
```
aws iam create-policy --policy-name Karpenter --policy-document "$(cat <<-EOM
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": [
                "eks:DescribeNodegroup",
                "eks:UpdateNodegroupConfig"
            ],
            "Effect": "Allow",
            "Resource": "*"
        }
    ]
}
EOM
)"
```
```
eksctl utils associate-iam-oidc-provider --region=${REGION} --cluster=${CLUSTER_NAME} --approve
eksctl create iamserviceaccount --cluster ${CLUSTER_NAME} \
--name default \
--namespace karpenter \
--attach-policy-arn "arn:aws:iam::${AWS_ACCOUNT_ID}:policy/Karpenter" \
--override-existing-serviceaccounts \
--approve

kubectl delete pods -n karpenter -l control-plane=karpenter
kubectl get pods -n karpenter
```

## Demo

### Apply YAML and Watch
```
wget https://raw.githubusercontent.com/ellistarn/karpenter-aws-demo/main/autoscaler.yaml

NODE_GROUP_ARN=$(aws eks describe-nodegroup --nodegroup-name demo --cluster-name $USER-karpenter-aws-demo --output json | jq -r ".nodegroup.nodegroupArn") \
envsubst < autoscaler.yaml | kubectl apply -f -

# Open in 5 separate terminals
watch -d 'kubectl get pods'
watch -d 'kubectl get nodes'
watch -d 'kubectl get metricsproducers.autoscaling.karpenter.sh demo -ojson | jq ".status.reservedCapacity"'
watch -d 'kubectl get horizontalautoscalers.autoscaling.karpenter.sh demo -ojson | jq ".status" | jq "del(.conditions)"'
watch -d 'kubectl get scalablenodegroups.autoscaling.karpenter.sh demo -ojson | jq "del(.status.conditions)"| jq ".spec, .status"'
```

### Scale the Pods and Nodes
```
wget https://raw.githubusercontent.com/ellistarn/karpenter-aws-demo/main/inflate.yaml

REPLICAS=2 envsubst < inflate.yaml | kubectl apply -f -
REPLICAS=10 envsubst < inflate.yaml | kubectl apply -f -
REPLICAS=30 envsubst < inflate.yaml | kubectl apply -f -
```

## Cleanup
```
eksctl delete cluster --name ${CLUSTER_NAME}
aws iam delete-policy --policy-arn arn:aws:iam::${AWS_ACCOUNT_ID}:policy/Karpenter
```
