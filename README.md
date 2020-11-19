# Karpenter AWS Demo (10-15 minutes to set up, 10 minutes per demo)

Karpenter allows users to configure different metrics producers that signal what to scale, and when to scale. Each of the folders in this repo show examples of how you can configure your cluster to work and see the scaling process in real-time on your own AWS account.

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

## Demo Options

You can continue the demo process from here by doing any of the available demos.

* [Queue Demo](https://github.com/ellistarn/karpenter-aws-demo/blob/main/queue/)
* [Capacity Reservation Demo](https://github.com/ellistarn/karpenter-aws-demo/blob/main/capacity-reservations)

## Cleanup

```bash

eksctl delete cluster --name ${CLUSTER_NAME}
aws iam delete-policy --policy-arn arn:aws:iam::${AWS_ACCOUNT_ID}:policy/Karpenter
```
