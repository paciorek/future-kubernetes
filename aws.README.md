## UNDER CONSTRUCTION

# follow instructions to install the AWS CLI and eksctl:
# https://docs.aws.amazon.com/eks/latest/userguide/getting-started-eksctl.html

# eks seems slower than gke

eksctl version

aws configure # enter AWS credentials 

eksctl create cluster \
--name future \
--region us-west-2 \
--nodegroup-name standard-workers \
--node-type t2.micro \
--nodes 2 \
--nodes-min 1 \
--nodes-max 4 \
--ssh-access \
--ssh-public-key ~/.ssh/ec2_rsa.pub \
--managed

eksctl create cluster \
--name future \
--region us-west-2 \
--nodegroup-name standard-workers \
--node-type t2.micro \
--nodes 2 \
--nodes-min 1 \
--nodes-max 4 \
--ssh-access \
--ssh-public-key ~/.ssh/ec2_rsa.pub \
--managed \
--vpc-public-subnets --vpc-public-subnets subnet-41d5f224,subnet-81591cf6

## subnets obtained from AWS console; need to be in same VPC

[✔]  using existing VPC (vpc-c71445a2) and subnets (private:[] public:[subnet-81591cf6 subnet-41d5f224])
[!]  custom VPC/subnets will be used; if resulting cluster doesn't function as expected, make sure to review the configuration of VPC/subnets

Ok, so this seems to have been created correctly


subnets are created dynamically and don't match what is seen at
https://us-west-2.console.aws.amazon.com/vpc/home?region=us-west-2#subnets:sort=SubnetId

not clear how to modify mapPublicIpOnLaunch
https://aws.amazon.com/blogs/containers/upcoming-changes-to-ip-assignment-for-eks-managed-node-groups/


paciorek@smeagol:/var/tmp/future-kubernetes (master)> eksctl create cluster \
> --name future \
> --region us-west-2 \
> --nodegroup-name standard-workers \
> --node-type t2.micro \
> --nodes 2 \
> --nodes-min 1 \
> --nodes-max 4 \
> --ssh-access \
> --ssh-public-key ~/.ssh/ec2_rsa.pub \
> --managed
[ℹ]  eksctl version 0.15.0
[ℹ]  using region us-west-2
[ℹ]  setting availability zones to [us-west-2a us-west-2d us-west-2c]
[ℹ]  subnets for us-west-2a - public:192.168.0.0/19 private:192.168.96.0/19
[ℹ]  subnets for us-west-2d - public:192.168.32.0/19 private:192.168.128.0/19
[ℹ]  subnets for us-west-2c - public:192.168.64.0/19 private:192.168.160.0/19
[ℹ]  using SSH public key "/accounts/gen/vis/paciorek/.ssh/ec2_rsa.pub" as "eksctl-future-nodegroup-standard-workers-ae:f0:5a:6b:3d:e7:ef:a3:e2:20:ed:40:66:ac:cb:5f" 
[ℹ]  using Kubernetes version 1.14
[ℹ]  creating EKS cluster "future" in "us-west-2" region with managed nodes
[ℹ]  will create 2 separate CloudFormation stacks for cluster itself and the initial managed nodegroup
[ℹ]  if you encounter any issues, check CloudFormation console or try 'eksctl utils describe-stacks --region=us-west-2 --cluster=future'
[ℹ]  CloudWatch logging will not be enabled for cluster "future" in "us-west-2"
[ℹ]  you can enable it with 'eksctl utils update-cluster-logging --region=us-west-2 --cluster=future'
[ℹ]  Kubernetes API endpoint access will use default of {publicAccess=true, privateAccess=false} for cluster "future" in "us-west-2"
[ℹ]  2 sequential tasks: { create cluster control plane "future", create managed nodegroup "standard-workers" }
[ℹ]  building cluster stack "eksctl-future-cluster"
[ℹ]  deploying stack "eksctl-future-cluster"



[ℹ]  building managed nodegroup stack "eksctl-future-nodegroup-standard-workers"
[ℹ]  deploying stack "eksctl-future-nodegroup-standard-workers"
[✖]  unexpected status "ROLLBACK_IN_PROGRESS" while waiting for CloudFormation stack "eksctl-future-nodegroup-standard-workers"
[ℹ]  fetching stack events in attempt to troubleshoot the root cause of the failure
[!]  AWS::EKS::Nodegroup/ManagedNodeGroup: DELETE_IN_PROGRESS
[✖]  AWS::EKS::Nodegroup/ManagedNodeGroup: CREATE_FAILED – "Nodegroup standard-workers failed to stabilize: [{Code: Ec2SubnetInvalidConfiguration,Message: One or more Amazon EC2 Subnets of [subnet-0d2944fddb54d2032, subnet-04bdc19cde2223ff0, subnet-0bad98cc3dd3bf7a5] for node group standard-workers does not automatically assign public IP addresses to instances launched into it. If you want your instances to be assigned a public IP address, then you need to enable auto-assign public IP address for the subnet. See IP addressing in VPC guide: https://docs.aws.amazon.com/vpc/latest/userguide/vpc-ip-addressing.html#subnet-public-ip,ResourceIds: [subnet-0d2944fddb54d2032, subnet-04bdc19cde2223ff0, subnet-0bad98cc3dd3bf7a5]}]"
[ℹ]  1 error(s) occurred and cluster hasn't been created properly, you may wish to check CloudFormation console
[ℹ]  to cleanup resources, run 'eksctl delete cluster --region=us-west-2 --name=future'
[✖]  waiting for CloudFormation stack "eksctl-future-nodegroup-standard-workers": ResourceNotReady: failed waiting for successful resource state
Error: failed to create cluster "future"

See here:
https://aws.amazon.com/blogs/containers/upcoming-changes-to-ip-assignment-for-eks-managed-node-groups/


eksctl delete cluster future
