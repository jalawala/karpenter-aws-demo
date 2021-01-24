# Karpenter AWS Demo (20 minutes)

This demo will explore how to use Karpenter to scale node groups using different metrics.

## Setup

### Environment

```bash
AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
CLUSTER_NAME=$USER-karpenter-aws-demo
REGION=us-west-2
```

### Cluster

```bash
eksctl create cluster \
--name ${CLUSTER_NAME} \
--version 1.18 \
--region ${REGION} \
--nodegroup-name karpenter-aws-demo \
--node-type m5.2xlarge \
--nodes 1 \
--nodes-min 1 \
--nodes-max 20 \
--managed
```

### nodegroup

```bash
eksctl create nodegroup \
--cluster eksworkshop \
--version 1.17 \
--region us-east-1 \
--name spot-mng-4vcp-16gb \
--instance-types m5.xlarge,m4.xlarge,m5a.xlarge,m5d.xlarge,m5n.xlarge,m5ad.xlarge,m5dn.xlarge \
--nodes 1 \
--nodes-min 1 \
--nodes-max 20 \
--managed  \
--asg-access \
--spot \
--node-labels "lc=spot,apps=noncritical,nodesize=spot4vcpu16gb" 
```

### nodegroup

```bash
eksctl create nodegroup \
--cluster eksworkshop \
--version 1.17 \
--region us-east-1 \
--name spot-mng-8vcp-32gb \
--instance-types m5.2xlarge,m4.2xlarge,m5a.2xlarge,m5d.2xlarge,m5n.2xlarge,m5ad.2xlarge,m5dn.2xlarge \
--nodes 1 \
--nodes-min 1 \
--nodes-max 20 \
--managed  \
--asg-access \
--spot \
--node-labels "lc=spot,apps=noncritical,nodesize=spot8vcpu32gb" 
```

### nodegroup

```bash
eksctl create nodegroup \
--cluster eksworkshop \
--version 1.17 \
--region us-east-1 \
--name od-mng-4vcp-16gb \
--instance-types m5.xlarge,m4.xlarge,m5a.xlarge,m5d.xlarge,m5n.xlarge,m5ad.xlarge,m5dn.xlarge \
--nodes 1 \
--nodes-min 1 \
--nodes-max 20 \
--managed  \
--asg-access \
--node-labels "lc=od,apps=critical,nodesize=od4vcpu16gb" 
```


### Karpenter Controller

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm upgrade --install cert-manager jetstack/cert-manager \
  --atomic \
  --create-namespace \
  --namespace cert-manager \
  --version v1.0.0 \
  --set installCRDs=true
helm upgrade --install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --atomic \
  --create-namespace \
  --namespace monitoring \
  --version 9.4.5 \
  --set alertmanager.enabled=false \
  --set grafana.enabled=false \
  --set kubeApiServer.enabled=false \
  --set kubelet.enabled=false \
  --set kubeControllerManager.enabled=false \
  --set coreDns.enabled=false \
  --set kubeDns.enabled=false \
  --set kubeEtcd.enabled=false \
  --set kubeScheduler.enabled=false \
  --set kubeProxy.enabled=false \
  --set kubeStateMetrics.enabled=false \
  --set nodeExporter.enabled=false \
  --set prometheus.enabled=false
kubectl apply -f https://raw.githubusercontent.com/awslabs/karpenter/v0.1.1/releases/aws/manifest.yaml
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

* [Queue Length](./queue-length/)
* [Reserved Capacity](./reserved-capacity)

## Cleanup

```bash
eksctl delete cluster --name ${CLUSTER_NAME}
aws iam delete-policy --policy-arn arn:aws:iam::${AWS_ACCOUNT_ID}:policy/Karpenter
```
