# Karpenter AWS Demo (20 minutes)

This demo will explore how to use Karpenter to scale node groups using different metrics.

## Setup

### Environment

```bash

CLOUD_PROVIDER=aws
AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
CLUSTER_NAME=$USER-karpenter-aws-demo
REGION=us-west-2
```

### Cluster

```bash

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

```bash

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

```bash

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
        },
        {
            "Action": [
                "autoscaling:DescribeAutoScalingGroups",
                "autoscaling:UpdateAutoScalingGroup"
            ],
            "Effect": "Allow",
            "Resource": "*"
        },
        {
            "Action": [
                "sqs:GetQueueAttributes",
                "sqs:GetQueueUrl",
                "sqs:ReceiveMessage",
                "sqs:DeleteMessage",
                "sqs:SendMessage"
            ],
            "Effect": "Allow",
            "Resource": "*"
        }
    ]
}
EOM
)"
```

```bash

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

## Demos

* [Queue Length](https://github.com/ellistarn/karpenter-aws-demo/blob/main/queue/)
* [Reserved Capacity](https://github.com/ellistarn/karpenter-aws-demo/blob/main/reserved-capacity)

## Cleanup

```bash

eksctl delete cluster --name ${CLUSTER_NAME}
aws iam delete-policy --policy-arn arn:aws:iam::${AWS_ACCOUNT_ID}:policy/Karpenter
```
