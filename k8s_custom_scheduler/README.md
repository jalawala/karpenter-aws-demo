# Custom Kube Scheduler with Karpenter

This repo contains a sample demo on how to run the custom kubernetes scheduler along with Karpneter.
The custom kube scheduler is designed to allow flexible placenent of pods across various nodegroups in a gievn ratio. This is similar to AWS ECS capacity provider straregy which allowws different weights for each capacity provider. Each service can specify its own scheduling strategy for the placement of tasks. Similary you can specify this strategy for each kubernetes deployment.

The open source Cluster Autoscaler for kubernetes does not support custom schedulers. Karpenter is a metric driven cluster autoscaler designed by AWS and it is open sourced.
This repo hosts a sample nginx deployment to demonstrate both custom kube scheduler and Karpneter.

## Prerequisites
Install Python3 is installed and run below command to deploy the required packages

```bash
curl https://bootstrap.pypa.io/get-pip.py -o get-pip.py
python3 get-pip.py --force-reinstall
pip3 install -r requirements.txt  
```
## Create EKS Cluster
Create an EKS cluster or re-use an existing cluster. Ensure that basic commands wor

```bash
kubectl get nodes 
```

## Create EKS Managed nodegroups

For this demo, create 1 EKS managed node group completely on On-demand and 2 using EC2 Spot

```bash
eksctl create nodegroup --cluster eksworkshop --version 1.17 --region us-east-1 --name od-mng-4vcp-16gb --instance-types m5.xlarge,m4.xlarge,m5a.xlarge,m5d.xlarge,m5n.xlarge,m5ad.xlarge,m5dn.xlarge --nodes 1 --nodes-min 1 --nodes-max 20 --managed  --asg-access --node-labels "lc=od,apps=critical,nodesize=od4vcpu16gb" 

eksctl create nodegroup --cluster eksworkshop --version 1.17 --region us-east-1 --name spot-mng-4vcp-16gb --instance-types m5.xlarge,m4.xlarge,m5a.xlarge,m5d.xlarge,m5n.xlarge,m5ad.xlarge,m5dn.xlarge --nodes 1 --nodes-min 1 --nodes-max 20 --managed  --asg-access --spot --node-labels "lc=spot,apps=noncritical,nodesize=spot4vcpu16gb"

eksctl create nodegroup --cluster eksworkshop --version 1.17 --region us-east-1 --name spot-mng-8vcp-32gb --instance-types m5.2xlarge,m4.2xlarge,m5a.2xlarge,m5d.2xlarge,m5n.2xlarge,m5ad.2xlarge,m5dn.2xlarge --nodes 1 --nodes-min 1 --nodes-max 20 --managed  --asg-access --spot --node-labels "lc=spot,apps=noncritical,nodesize=spot8vcpu32gb"
```

Get the node group ARNs and deploy the Karpenter resources


```bash
cd karpenter-aws-demo/k8s_custom_scheduler \
OD_NODE_GROUP_ARN=$(aws eks describe-nodegroup --nodegroup-name od-mng-4vcp-16gb --cluster-name eksworkshop --output json | jq -r ".nodegroup.nodegroupArn") \
echo "OD_NODE_GROUP_ARN = $OD_NODE_GROUP_ARN"  \
SPOT_NODE_GROUP1_ARN=$(aws eks describe-nodegroup --nodegroup-name spot-mng-4vcp-16gb --cluster-name eksworkshop --output json | jq -r ".nodegroup.nodegroupArn") \
echo "SPOT_NODE_GROUP1_ARN = $SPOT_NODE_GROUP1_ARN"  \
SPOT_NODE_GROUP2_ARN=$(aws eks describe-nodegroup --nodegroup-name spot-mng-8vcp-32gb --cluster-name eksworkshop --output json | jq -r ".nodegroup.nodegroupArn") \
echo "SPOT_NODE_GROUP2_ARN = $SPOT_NODE_GROUP2_ARN"  \
envsubst < karpenter_resources.yaml | kubectl apply -f -

```

## Create port forward for the prometheus service and open it in brower

```bash
kubectl -n karpenter port-forward prometheus-karpenter-monitor-0 8080:9090
```


## Configuring the Deployment for the Kube Scheduler
Add the following annotation to the deployment in nginx.yaml
Note that replica is set to 5 initially.

```bash
kind: Deployment
metadata:
  name: nginx
  annotations:
    UseCustomKubeScheduler: 'true'
    CustomPodScheduleStrategy: 'nodesize=od4vcpu16gb,base=1,weight=0:nodesize=spot4vcpu16gb,weight=1:nodesize=spot8vcpu32gb,weight=1'
  labels:
    ...  
```
The *CustomPodScheduleStrategy* specify how pods within a deployment should be spread

## Deploy the Sample nginx deployment. 

```bash
cd karpenter-aws-demo/k8s_custom_scheduler
kubectl apply -f nginx.yaml
```
## Run the Kube Custom Scheduler 

Open a new terminal and run the kube scheduler on the command line.
Note this will deploted as pod in the cluster eventually.

```bash
cd karpenter-aws-demo/k8s_custom_scheduler
./CustomKubeScheduler.py
```
Note Kubescheduker runs main loop every 10 sec. To help in debugging, it waits for any keyboard press from the user before running the next loop.
So just press any letetr and enter to run the next loop

Note that kube scheduler deploy the 5 pods in this ratio

```bash
namespace=default deploymentName=nginx CustomSchedulingData={'nodesize=od4vcpu16gb': 1, 'nodesize=spot4vcpu16gb': 2, 'nodesize=spot8vcpu32gb': 2}
```

Now change the replica from 5 to 50 nginx.yamla and apply it again

```bash
cd karpenter-aws-demo/k8s_custom_scheduler
kubectl apply -f nginx.yaml
```
Now the distribution should look like below. This mean scaling is required.

```bash
namespace=default deploymentName=nginx CustomSchedulingData={'nodesize=od4vcpu16gb': 1, 'nodesize=spot4vcpu16gb': 24, 'nodesize=spot8vcpu32gb': 25}
```
Check if Karpenter scaled the nodes in the nodelabels 

```bash
Available node with Label: nodesize=od4vcpu16gb i=1 node=ip-192-168-73-104.ec2.internal
Available node with Label: nodesize=spot4vcpu16gb i=1 node=ip-192-168-14-154.ec2.internal
Available node with Label: nodesize=spot4vcpu16gb i=2 node=ip-192-168-61-28.ec2.internal
Available node with Label: nodesize=spot8vcpu32gb i=1 node=ip-192-168-51-161.ec2.internal
```
We can see Karpenter scales up/down the nodes continousley eben though there is no changes to the pods

## Run Custom Scheduler as Deployment

This section is TBD

### Step0 :  Ensure CA is not running and manually scale ASGs with instances needed for your workload
 kubectl delete -f ~/environment/cluster-autoscaler/cluster_autoscaler.yml 

### Step1 :  Run the customer Scheduler locally on your development machine

Setup the kubeconfig on your machine and ensure basic command like 'kubectl get nodes' work

Ensure to have below lines enabled in Ec2SpotCustomScheduler.py

config.load_kube_config()
#config.load_incluster_config()

chmod +x Ec2SpotCustomScheduler.py
./Ec2SpotCustomScheduler.py

### Step2 :  Run the customer Scheduler in your K8S cluster as a pod - Create an ECR Repo

aws ecr create-repository \
    --repository-name ec2spotcustomscheduler \
    --image-scanning-configuration scanOnPush=true
    
### Step3 :  Build customer Scheduler container and push it to ECR

cd /home/ec2-user/environment/ec2-spot-workshops/workshops/ec2-spot-game-day/k8s_custom_scheduler

aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 000474600478.dkr.ecr.us-east-1.amazonaws.com

Ensure to have below lines enabled in Ec2SpotCustomScheduler.py

#config.load_kube_config()
config.load_incluster_config()


docker build --no-cache -t ec2spotcustomscheduler .
docker tag ec2spotcustomscheduler:latest 000474600478.dkr.ecr.us-east-1.amazonaws.com/ec2spotcustomscheduler:latest
docker push 000474600478.dkr.ecr.us-east-1.amazonaws.com/ec2spotcustomscheduler:latest
 
### Step4 :  Build and Deploy the customer Scheduler onto OnDemand Instances
Update the docker image path in Ec2SpotCustomScheduler.yaml. Please note it will be deployed on the OnDemand instances to keep it running all the time.
NOTE: A ServiceAccount with necessary permissions needs to added to below Yaml to access the K8S objects.
kubectl apply  -f Ec2SpotCustomScheduler.yaml 

To run the custom scheduler manually on the devlopment machine
### Step5 :  Create a deployement with the custom scheduling information

Refer to sample file nginx.yaml

Add the below annotation to the deployment object
  annotations:
    Ec2SpotK8SCustomScheduler: 'true'  # This enables the custome scheduling for a deployment
    OnDemandBase: '1'  # number of min pods to be running on OnDemand, similar to ASG
    OnDemandAbovePercentage: '50' # % of pods beyond the base number of OnDemand pods
    SpotASGName: 'Ec2SpotEKS4-Ec2SpotNodegroup1' # NOT NEEDED for now. We can use it later if we want to scale ASG  
    OnDemandASGName: 'Ec2SpotEKS4-ODNodegroup1' # NOT NEEDED for now. We can use it later if we want to scale ASG  
    
    
    Add below to the pod spec 
    schedulerName: "Ec2SpotK8sScheduler"

    kubectl apply -f nginx.yaml
    

### Step6 : Monitor the logs for the custom scheduler
kubectl logs -f ec2spotcustomscheduler-78b6c6c989-bzn5x


### Step7 : Try changing the pod distribution by changing OnDemandBase, OnDemandAbovePercentage and replica

export EDITOR=vim
kubectl edit deploy nginx

check out the custom scheduler logs also check 'kubectl get pods'


### TODO List - Integration with K8S Cluster Autoscaler (CA)

1. K8S CA DOES NOT scale instances for the pods assigned to with custom scheduler. CA takes only default K8S scheduler into account.
2. Even the ASG is scaled manually and place the pods using custom scheduler, CA will terminate these nodes and pods.
3. One option is to use ASG scaling policies(ex: target tracking) instead of CA for scaling cluster

## Setup the Karpenter Resources

### Get the node group ARNs

Get the list of node groups in the cluster

```bash
aws eks list-nodegroups --cluster eksworkshop
```


# Manually open in 5 separate terminals
   "od-mng-4vcp-16gb",
        "spot-mng-4vcp-16gb",
        "spot-mng-8vcp-32gb"
        
        
```bash
watch 'kubectl get pods -n karpenter-custom-kube-scheduler-demo'
watch 'kubectl get nodes'
watch -d 'kubectl get metricsproducers.autoscaling.karpenter.sh od-mng-4vcp-16gb -n karpenter-custom-kube-scheduler-demo -ojson | jq .status.reservedCapacity'
watch -d 'kubectl get horizontalautoscalers.autoscaling.karpenter.sh od-mng-4vcp-16gb -n karpenter-custom-kube-scheduler-demo -ojson | jq ".status" | jq del\(.conditions\)'
watch -d 'kubectl get scalablenodegroups.autoscaling.karpenter.sh od-mng-4vcp-16gb -n karpenter-custom-kube-scheduler-demo -ojson | jq "del(.status.conditions)"| jq ".spec, .status"'
```

```bash
watch -d 'kubectl get metricsproducers.autoscaling.karpenter.sh spot-mng-4vcp-16gb -n karpenter-custom-kube-scheduler-demo -ojson | jq .status.reservedCapacity'
watch -d 'kubectl get horizontalautoscalers.autoscaling.karpenter.sh spot-mng-4vcp-16gb -n karpenter-custom-kube-scheduler-demo -ojson | jq ".status" | jq del\(.conditions\)'
watch -d 'kubectl get scalablenodegroups.autoscaling.karpenter.sh spot-mng-4vcp-16gb -n karpenter-custom-kube-scheduler-demo -ojson | jq "del(.status.conditions)"| jq ".spec, .status"'
```


```bash
watch 'kubectl get pods -n karpenter-custom-kube-scheduler-demo'
watch 'kubectl get nodes'
watch -d 'kubectl get metricsproducers.autoscaling.karpenter.sh spot-mng-8vcp-32gb -n karpenter-custom-kube-scheduler-demo -ojson | jq .status.reservedCapacity'
watch -d 'kubectl get horizontalautoscalers.autoscaling.karpenter.sh spot-mng-8vcp-32gb -n karpenter-custom-kube-scheduler-demo -ojson | jq ".status" | jq del\(.conditions\)'
watch -d 'kubectl get scalablenodegroups.autoscaling.karpenter.sh spot-mng-8vcp-32gb -n karpenter-custom-kube-scheduler-demo -ojson | jq "del(.status.conditions)"| jq ".spec, .status"'
```